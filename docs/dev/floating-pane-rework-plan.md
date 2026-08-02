# Floating-pane rework plan

Rework of the per-tab floating pane (introduced in commit `09da2d20b`, branch
`floating-pane-experiment`) from a hacky prototype into a principled feature.

This document is the single source of truth for the rework. Each phase is
executed independently (one commit each), with a review checkpoint between
phases. An executing agent should read this whole file, then implement **only
the phase it is assigned**, keeping the diff scoped to that phase.

---

## Guiding principle

A floating pane is exactly three things, and every current hack comes from
faking one of them instead of modelling it:

1. **One window excluded from tiling** (it does not participate in the layout's
   group geometry or in group navigation).
2. **With independent geometry** (a centred sub-rectangle, sized normal/expanded).
3. **Rendered on a top z-layer** (painted above all tiled windows, hit-tested first).

Model those three properties directly; delete the machinery that fakes them.

---

## How the relevant machinery works (reference)

Draw order / z-order:
- The C render loop paints windows in `tab->windows[]` **array order**; the last
  entry paints on top (painter's algorithm, no depth test).
  `kitty/child-monitor.c:899` `render_prepared_os_window`, loop at `:910`.
- Array order is set by attach order: `add_window` appends
  (`kitty/state.c:378`), `detach_window`/`attach_window` (`kitty/state.c:474`,
  `:502`) remove/re-append. `detach`+`attach` is the only way to move a window
  to the array end today.
- Geometry reaches C via `Window.set_geometry` → `set_window_render_data`
  (`kitty/window.py:1050`, `kitty/state.c:1199`); stored in
  `window->render_data.geometry`. `WindowGeometry` = content px bounds
  (`left/top/right/bottom`) + `xnum/ynum` cells + `spaces` (px reserved for
  margin+border+padding per side). `kitty/types.py:49`, `Edges` at `:35`.
- `viewport_for_window(os_window_id)` → `(central, tab_bar, vw, vh, cell_width,
  cell_height)`; `central` is a `Region(left,top,right,bottom,width,height)` px
  drawing canvas after the tab bar. `kitty/state.c:1038`.

Visibility / layout:
- `Layout.__call__` → `_set_dimensions` → `update_visibility` → `do_layout`.
  `kitty/layout/base.py:404`.
- `update_visibility` (`base.py:391`): a window is visible iff it is
  `all_windows.active_window`, or it is a group leader and the layout is not
  `only_active_window_visible`. Stack sets `only_active_window_visible = True`
  (`kitty/layout/stack.py:15`), so under stack **only the single window equal to
  `active_window` is visible**.
- `set_visible_in_layout` (`kitty/window.py:1007`) only touches C / refreshes
  when the value actually changes (idempotent).
- Typing does **not** relayout. `relayout()` is called only on structural events
  (add/remove/move window, layout change, resize, floating toggles).

Borders:
- `Borders.__call__` (`kitty/borders.py:81`) builds a flat list of `Border`
  rects and ships them via `set_borders_rects`. Full-border mode iterates
  `all_windows.iter_all_layoutable_groups(only_visible=True)` (`:95`) — which
  **excludes the float** — so the float never gets a frame. `add_borders`
  (`:41`) computes the 4 edge rects from `wg.geometry` + padding + border width.
- Borders paint before window cells (`child-monitor.c:904`), so a float border
  rect sits correctly around the (top-painted) float cells.

Window groups vs overlays:
- Overlays share one `WindowGeometry` per `WindowGroup`
  (`kitty/window_list.py:141`) → cannot have independent geometry. The float is
  therefore correctly modelled as its **own** group, excluded from tiling — not
  as an overlay. Overlay/group machinery is reusable only for the "last attached
  wins" z-order idea, which Phase 3 replaces anyway.

---

## Root causes of the four reported problems

1. **"Iterating from back / hacks"** — z-order is faked by mutating the C window
   array. `_raise_floating_window` (`kitty/tabs.py:599`) does `detach_window` +
   `attach_window` to shove the float to the array end. This desyncs Python
   `groups`/`all_windows` order from the C array, which forced: the back-to-front
   loop in `kitty/mouse.c:1064`, the re-raise in `_add_window` (`tabs.py:940`),
   and the defensive `suppress(Exception)` + history fallbacks in
   `active_group`/`active_window`/`active_group_main` (`window_list.py:409-439`).

2. **Stack-layout flicker on typing** — a hide-then-show race. When the float is
   focused, `active_window` *is* the float, and the float is excluded from
   `iter_windows_with_visibility()`. Under stack (`only_active_window_visible`),
   `update_visibility` therefore hides **every** tiled window, and ad-hoc
   `set_visible_in_layout(True)` calls in `_ensure_floating_window` (`tabs.py:528`)
   and `_apply_floating_geometry` (`tabs.py:597`) re-show one. Hide→show each
   pass, and the C `num_of_visible_windows` count flips (changing the
   single-window bg/border path, `child-monitor.c:909`) → flicker.

3. **No border** — the float is deliberately excluded from the border pass
   (`borders.py:95`, see above). Nothing ever draws its frame.

4. **"Data structures tacked on"** — four parallel fields on `Tab`
   (`floating_window_id`, `floating_rect`, `floating_enabled`, `floating_size_mode`,
   `tabs.py:222-225`) plus a global `layout_exclude_window_id` on `WindowList`
   threaded through ~8 methods with an empty-filter fallback hack
   (`window_list.py:183`). Dead code: `move_group_to_end` (`window_list.py:599`,
   never called). Half-baked persistence: floating state is *written* in
   `serialize_state`/`list_tabs` (`tabs.py:399`, `:1697`) but **never restored** —
   `startup` ignores it, so the `__init__` restore branch (`tabs.py:242`) is dead.

---

## Decisions (locked)

- **Z-order:** first-class C flag. Add a `floating`/z_index field to the C
  `Window` struct; render tiled then floating in two passes; delete
  detach/attach, the `_add_window` re-raise, and rewrite `mouse.c` as intentional
  topmost-first hit-testing. Removes the Python/C desync entirely.
- **Focus:** independently focusable. The float behaves like a WM floating
  window — focus can move in/out of it; tiled windows underneath can be focused
  by click while the float stays visible on top. Drop the "focus locked to
  float while enabled" logic.
- **Persistence:** complete the restore. Serialize `FloatingPane` and restore it
  in `startup()`; keep `take_over_from` working; remove the dead `__init__`
  restore branch.

---

## Phases

Each phase = one commit, then STOP for human review. Keep each diff scoped.

### Phase 1 — Encapsulate state (pure refactor, no behavior change)

Goal: remove the "tacked on" feel. No user-visible behavior change.

- Add a `FloatingPane` value object (dataclass) holding `window_id: int`,
  `enabled: bool`, `size_mode: FloatingSizeMode`, `rect: tuple[int,int,int,int] |
  None` (rect in cells). Move the pure geometry helpers onto it or a small
  helper: `_default_floating_rect`, `_expanded_floating_rect`,
  `_floating_rect_for_mode`, `_clamp_floating_rect`. `Tab.floating: FloatingPane
  | None` replaces the four loose fields.
- Update all readers/writers of the four fields (`get_floating_window`,
  `is_floating_window`, `clear_floating_window`, `set_floating_window`,
  `_ensure_floating_window`, toggles, `take_over_from`, `serialize_state`,
  `list_tabs`, `active_window_changed`, `set_active_window`, `_add_window`,
  `remove_window`) to go through the new object.
- Rename `WindowList.layout_exclude_window_id` → `floating_window_id`; add two
  clean accessors: `layoutable_groups(only_visible=False)` and
  `navigable_group_indices()`. Drop the empty-filter fallback at
  `window_list.py:183` (handle "only the float exists" explicitly — a tab whose
  only window is the float should still render the float).
- Delete the unused `move_group_to_end` (`window_list.py:599`).

Acceptance: `python -m kitty +launch` builds; toggling the float on/off and the
size toggle behave exactly as before; `git diff` reads as a structural refactor
(no logic change). Existing `kitty_tests` pass.

### Phase 2 — Fix the flicker at its source

Goal: eliminate stack-layout flicker; make underlying window stay put.

- Add `WindowList.layout_active_window`: the active window **ignoring** the
  float. If `active_window` is the float, fall back to the most-recent
  non-floating group (use `active_group_history`), else the active window.
- Change `update_visibility` (`kitty/layout/base.py:391`) to compute tiled
  visibility from `layout_active_window` instead of `active_window`. Make the
  float's visibility one deterministic decision driven by `FloatingPane.enabled`
  (enabled ⇒ visible). Remove the ad-hoc `set_visible_in_layout(True)`
  compensations in `_ensure_floating_window` and `_apply_floating_geometry`.
- Ensure the visible-window count is stable across frames (underlying window +
  float both visible) so the C single-window path does not toggle.

Acceptance: with a stack layout and a focused float, typing into the float shows
**no flicker**; the underlying tiled window remains visible beneath the float;
switching size mode does not flicker.

### Phase 3 — First-class z-order (removes the array-mutation hacks)

Goal: real top z-layer; delete the detach/attach dance and its downstream hacks.

- Add a `bool floating` (or a small `z_index`) to the C `Window` struct
  (`kitty/state.h:` Window definition) and a way to set it from Python (extend
  `set_window_render_data`, or add a dedicated setter exported via
  `fast_data_types`). Python sets it when a window becomes / stops being the
  float.
- `render_prepared_os_window` (`kitty/child-monitor.c:910`): render in two
  passes — first all visible non-floating windows, then all visible floating
  windows. Preserve `is_active_window`/title-bar draw semantics and the
  `num_of_visible_windows == 1` single-window handling.
- Delete `_raise_floating_window` (`tabs.py:599`), the detach/attach calls, and
  the re-raise block in `_add_window` (`tabs.py:940-942`).
- Rewrite the `mouse.c` change (`mouse.c:1064`) as intentional topmost-first
  hit-testing: test visible floating windows before tiled windows and return the
  first hit; drop the reversed-index loop.
- Now that Python `groups`/`all_windows` order and the C array agree again,
  revert the defensive hardening in `active_group`/`active_window`/
  `active_group_main` (`window_list.py:409-439`) back toward the simpler
  original — but only after confirming no other caller relied on the fallbacks.

Acceptance: opening a new split / window while the float is enabled no longer
covers the float (no re-raise needed); clicking the float vs. a tiled window
hits the right one; no detach/attach anywhere in the floating code path.

### Phase 4 — Border around the float

Goal: give the float a visible frame.

- Add a `WindowList.floating_group` accessor returning the float's `WindowGroup`
  (or None).
- In `Borders.__call__` (`kitty/borders.py:81`), after the tiled border pass,
  if the tab has an enabled float, append its border rects via `add_borders`
  using the active border color (MVP). The float geometry's `spaces` already
  reserves the border width, so it frames cleanly.
- Optional: add a `floating_border_color` option (definition.py) for a distinct
  frame; default to the active border color if unset.

Acceptance: the float shows a clear border distinguishing it from the tiled
windows underneath, in both normal and expanded size modes.

### Phase 5 — Focus model + persistence

Goal: WM-style independent focus; working session restore.

- Independently focusable: drop the "focus locked to float while enabled" block
  in `set_active_window` (`tabs.py:1088-1094`) and the redirect in
  `active_window_changed` (`tabs.py:470`). Toggling the float off moves focus to
  `layout_active_window`. Clicking a tiled window focuses it with the float still
  visible on top.
- Persistence: serialize the `FloatingPane` (already partly present) and
  **restore** it in `startup()` (`tabs.py:325`) — re-adopt the float window and
  its enabled/size_mode/rect. Keep `take_over_from` (`tabs.py:293`) copying the
  object. Remove the dead `__init__` restore branch (`tabs.py:242`).

Acceptance: click-to-focus works for tiled windows while the float is up;
toggling off returns focus sensibly; saving and restoring a session (and
detaching/moving a tab) preserves the float, its size mode, and enabled state.

### Phase 6 — Tests + docs

- `kitty_tests`: float excluded from group navigation and tiling; stack
  visibility keeps the underlay visible (no hidden underlay, no flicker
  proxy via visible-count); border rects present for the float; z-order correct
  after opening a split; serialize→restore round-trip; focus in/out.
- Flesh out the thin `docs/layouts.rst` floating-pane section
  (`docs/layouts.rst:32`): describe behavior, size modes, keybindings
  (`kitty_mod+i`, `kitty_mod+alt+i`), and the border option if added.

Acceptance: new tests pass; docs describe the final behavior accurately.

---

## Cross-cutting notes / risks

- `startup()` window re-adoption must not fight the layout-state restore — order
  matters; restore the float after the tiled groups exist.
- The two-pass render must preserve `is_active_window` and title-bar draw
  semantics and the single-window background path.
- When reverting `window_list.py` hardening in Phase 3, confirm no non-floating
  caller depended on the defensive fallbacks (grep callers of `active_group`,
  `active_window`, `active_group_main`).
- Keep every phase's diff scoped so regressions bisect cleanly to a phase.

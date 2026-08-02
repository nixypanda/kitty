# AGENTS

This repo was built from source on macOS using Nix.

## Nix dev shell

- A flake wrapper was added to pin nixpkgs: `flake.nix`.
- `shell.nix` was adjusted to avoid the deprecated `darwin.apple_sdk.frameworks` alias and instead use `apple-sdk_15` when available.
- Enter the dev shell:

```bash
nix develop --extra-experimental-features 'nix-command flakes'
```

## Build

```bash
nix develop --extra-experimental-features 'nix-command flakes' --command make
```

## Tests

```bash
nix develop --extra-experimental-features 'nix-command flakes' --command make test
```

Expected failures in Nix dev shell:

- `kitty_tests.shell_integration.ShellIntegration.test_bash_integration`
- `kitty_tests.shell_integration.ShellIntegrationWithKitten.test_bash_integration`

Reason: bash reports `complete: not a shell builtin` under the Nix shell. Do not try to fix these.

## App bundle (macOS)

Man pages must be generated before building the app bundle:

```bash
nix develop --extra-experimental-features 'nix-command flakes' --command make docs
nix develop --extra-experimental-features 'nix-command flakes' --command make app
```

The bundle is created at `kitty.app` in the repo root.

Run the built app from CLI:

```bash
./kitty.app/Contents/MacOS/kitty --version
```

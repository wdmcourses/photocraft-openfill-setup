# PhotoCraft OpenFill Setup

This repository holds two independent agent instructions for a portable, self-contained PhotoCraft + OpenFill bundle on Windows, Linux, and macOS. `setup.md` is the installer; `path.md` is an optional patch that only adds the Reference layer.

## Files

- **`setup.md`** — installs the portable bundle from scratch into an empty folder. This is all most users need.
- **`path.md`** — optional: patches an existing OpenFill package to add the Reference layer select. It changes exactly eight files and keeps a backup for rollback.

## Usage

For the portable app alone, put `setup.md` in the empty target folder and mention it:

```text
@setup.md
```

Only if you also want the Reference layer, place `path.md` in the portable folder next to `OpenFill/` after setup and mention it:

```text
@path.md
```

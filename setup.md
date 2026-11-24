# setup.md — portable PhotoCraft + OpenFill (Windows, Linux, macOS)

`ROOT` is the folder that contains this file. Build everything inside `ROOT`; create no app
data outside it. The optional Linux `.desktop` shortcut is the only external file (section 5).
Do not download ComfyUI or models, and do not launch the app at the end.

The result is one self-contained folder: PhotoCraft, OpenFill, a private CPython 3.12 with
Pillow and Tk, a private profile, and a pinnable launcher. No system Python, no system
packages, no app data in the real APPDATA / XDG directories / Library.

## 1. Platform

Match the OS and architecture to one row:

| OS | arch | Python triple | PhotoCraft asset |
|---|---|---|---|
| Windows | x64 | `x86_64-pc-windows-msvc` | `*-windows-x64-portable.zip` |
| Windows | arm64 | `aarch64-pc-windows-msvc` | `*-windows-arm64-portable.zip` |
| Linux | x86_64 | `x86_64-unknown-linux-gnu` | `*-linux-x86_64.AppImage` |
| Linux | aarch64 | `aarch64-unknown-linux-gnu` | `*-linux-aarch64.AppImage` |
| macOS | Intel | `x86_64-apple-darwin` | `*-macos-universal.dmg` |
| macOS | Apple Silicon | `aarch64-apple-darwin` | `*-macos-universal.dmg` |

On Linux use the glibc build. If the machine does not match any row, stop and report.

## 2. PhotoCraft

Download the newest release asset for this row from
`https://github.com/storytold/photocraft/releases/latest` into `ROOT/PhotoCraft/`.

- **Windows:** unzip it; `PhotoCraft/photocraft.exe` must exist. Delete the zip.
- **Linux:** `chmod +x` the AppImage, then extract it without FUSE:
  `cd ROOT/PhotoCraft && ./*.AppImage --appimage-extract`.
  The editor is `PhotoCraft/squashfs-root/AppRun`. Delete the AppImage. If
  `squashfs-root/*.png` or `squashfs-root/.DirIcon` exists, copy it to
  `PhotoCraft/icon.png`.
- **macOS:** `mkdir -p ROOT/profile/Temp/dmg`, mount the DMG read-only there
  (`hdiutil attach -nobrowse -readonly -mountpoint ...`), copy `PhotoCraft.app` into
  `ROOT/PhotoCraft/`, detach, then run
  `xattr -dr com.apple.quarantine ROOT/PhotoCraft/PhotoCraft.app`. Delete the DMG.

## 3. OpenFill

From `https://github.com/beckettharriman/openfill/releases` take the newest release whose tag
starts with `photocraft-v`; its asset is `openfill-photocraft-<ver>.zip`. Extract it into
`ROOT/OpenFill/` and keep the versioned folder (`openfill-photocraft-<ver>/`). Do not run its
install scripts: the companion adds its PhotoCraft menu when it starts.

## 4. Private Python

Download the newest CPython **3.12** `install_only` archive for this row from
`https://github.com/astral-sh/python-build-standalone/releases/latest`:
asset `cpython-3.12.*-<Python triple>-install_only.tar.gz` (never `freethreaded`). If 3.12 is
absent, use 3.13. Extract it into `ROOT`; the archive creates `ROOT/python/`.

- Windows interpreter: `ROOT\python\python.exe`
- Linux/macOS interpreter: `ROOT/python/bin/python3`

Install Pillow into it, with no user site and no pip cache:

- **Windows:** in one shell: `set "PYTHONNOUSERSITE=1"` and then
  `ROOT\python\python.exe -m pip install --no-cache-dir --disable-pip-version-check pillow`
- **Linux/macOS:**
  `PYTHONNOUSERSITE=1 ROOT/python/bin/python3 -m pip install --no-cache-dir --disable-pip-version-check pillow`

Check it: `... -c "import tkinter, PIL; print('python ok')"`. If Tk is missing, stop and
report: the panel and the Reference select need it.

## 5. Launchers

Every launcher sets these variables to absolute paths inside `ROOT`:

- `PHOTOCRAFT_BIN`
  - Windows: `ROOT/PhotoCraft/photocraft.exe`
  - Linux: `ROOT/PhotoCraft/squashfs-root/AppRun`
  - macOS: the executable inside `ROOT/PhotoCraft/PhotoCraft.app/Contents/MacOS` (use its
    real name; current builds use `PhotoCraft`)
- `PYTHONPATH=ROOT/OpenFill/openfill-photocraft-<ver>`
- `PYTHONNOUSERSITE=1`
- Profile redirection:
  - **Windows:** `APPDATA=ROOT/profile/AppData/Roaming`,
    `LOCALAPPDATA=ROOT/profile/AppData/Local`, `TEMP=TMP=ROOT/profile/Temp`
  - **Linux:** `HOME=ROOT/profile/home`, `XDG_CONFIG_HOME=ROOT/profile/config`,
    `XDG_DATA_HOME=ROOT/profile/data`, `XDG_CACHE_HOME=ROOT/profile/cache`,
    `XDG_RUNTIME_DIR=ROOT/profile/run` (mode 700), `TMPDIR=ROOT/profile/tmp`
  - **macOS:** `HOME=ROOT/profile/home`, `CFFIXED_USER_HOME=ROOT/profile/home`, the same
    `XDG_*`, `TMPDIR`

Create the profile directories before starting.

Run the companion:

- **Windows:** `ROOT\python\pythonw.exe -s -m openfill_photocraft`
- **Linux/macOS:** `ROOT/python/bin/python3 -s -m openfill_photocraft`

### Windows

Write `launcher.cmd` using `%~dp0` (so the folder can move) and the variables above. Create
`PhotoCraft.lnk` with target `launcher.cmd`, working directory `ROOT`, icon
`PhotoCraft\photocraft.exe,0` (PowerShell `WScript.Shell`).

### Linux

Write `launcher.sh` with `DIR=$(cd "$(dirname "$0")" && pwd)` and `exec` the private Python;
`chmod +x` it. Write `PhotoCraft.desktop`:

```
[Desktop Entry]
Type=Application
Name=PhotoCraft
Exec="<abs>/launcher.sh"
Path=<abs>
Icon=<abs>/PhotoCraft/icon.png
Terminal=false
Categories=Graphics;
```

Use `Icon=applications-graphics` when no icon was extracted. For pinning, copy the `.desktop`
to `~/.local/share/applications/` (a shortcut only) and pin it from the app menu. It holds
absolute paths; recreate it if the folder moves.

### macOS

Write executable `PhotoCraft.command` that sets the same environment and `exec`s the private
Python. For the Dock, create `PhotoCraft.app`:

- `Contents/Info.plist` with `CFBundleExecutable=PhotoCraft`,
  `CFBundleIdentifier=local.openfill.photocraft`,
  `CFBundleName=PhotoCraft`, `CFBundlePackageType=APPL`,
  `CFBundleVersion=1.0`; if you copied an `.icns` from
  `ROOT/PhotoCraft/PhotoCraft.app/Contents/Resources` to `Contents/Resources/AppIcon.icns`,
  also set `CFBundleIconFile=AppIcon`.
- `Contents/MacOS/PhotoCraft` (executable) runs
  `exec "$(cd "$(dirname "$0")/../../.." && pwd)/PhotoCraft.command"`.
- `xattr -dr com.apple.quarantine` on the wrapper.

## 6. Verify

- The private Python imports `tkinter` and `PIL`.
- Expanding the launcher variables keeps every path inside `ROOT`.
- The shortcut / `.desktop` / `.app` target exists and points at the launcher.
- No file was created outside `ROOT` (the optional Linux `.desktop` copy is the only shortcut).

Do not start the app.

## Report

OS and architecture, PhotoCraft and OpenFill versions, Python version, folder layout, launcher
path, the exact start command, and how to pin (taskbar / app menu / Dock).

## Expected layout

```
ROOT/
  PhotoCraft/                    editor (exe / squashfs-root/AppRun / PhotoCraft.app)
  OpenFill/
    openfill-photocraft-<ver>/
  python/                        private CPython 3.12 + Pillow + Tk
  profile/                       APPDATA / XDG / Library / temp
  launcher.cmd                   Windows
  PhotoCraft.lnk                 Windows
  launcher.sh                    Linux
  PhotoCraft.desktop             Linux
  PhotoCraft.command             macOS
  PhotoCraft.app                 macOS (Dock)
```

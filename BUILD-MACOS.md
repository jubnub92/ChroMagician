# Building ChroMagician on macOS (Apple Silicon)

Unofficial, community-tested notes. Tested on a 1st Edition Chromatic with
ChroMagic 1.0.1.

> ⚠️ **The Transparent Chromatic (released Oct 2026) is not working with
> ChroMagic 1.0.1:** cartridges were unreadable after install in testing.
> Firmware installation can leave a device unusable. Back up first, and try the
> read-only commands before installing anything.

`<repo>` below means the folder where you cloned this repository.

## Requirements
- Xcode with command line tools
- Flutter / Dart
- Rust (`rustup`)
- [Homebrew](https://brew.sh)
- `esptool` 4.12.0 and `openFPGALoader` 1.1.1 on PATH (the versions the
  Linux/Windows installers use)

## Build the app
```bash
cd <repo>/flutter
dart run tool/build_desktop.dart --catalog-url https://chromagic.org
```
Output: `<repo>/flutter/build/macos/Build/Products/Release/chromatic_pc_backup.app`

## Install the flashing tools
```bash
mkdir -p ~/chromagician-tools/bin && cd ~/chromagician-tools
```

**esptool 4.12.0** (official macOS arm64 release):
```bash
curl -LO https://github.com/espressif/esptool/releases/download/v4.12.0/esptool-v4.12.0-macos-arm64.tar.gz
tar xzf esptool-v4.12.0-macos-arm64.tar.gz
xattr -dr com.apple.quarantine ~/chromagician-tools
ln -sf ~/chromagician-tools/esptool-macos-arm64/esptool ~/chromagician-tools/bin/esptool
```

**openFPGALoader 1.1.1** (built from source with GWU2X support):
```bash
brew install --only-dependencies openfpgaloader
brew install cmake pkg-config zlib gzip
curl -L -o ofl.tar.gz https://codeload.github.com/trabucayre/openFPGALoader/tar.gz/c3a1232f8730dbde42453ab96d0275b29a384271
tar xzf ofl.tar.gz
cmake -S openFPGALoader-c3a1232f8730dbde42453ab96d0275b29a384271 -B ofl-build \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$HOME/chromagician-tools/ofl"
cmake --build ofl-build --parallel
cmake --install ofl-build
ln -sf ~/chromagician-tools/ofl/bin/openFPGALoader ~/chromagician-tools/bin/openFPGALoader
```
The configure step should print `Gowin GWU2X support enabled`.

Put them on PATH and check:
```bash
export PATH="$HOME/chromagician-tools/bin:$PATH"
esptool version              # 4.12.0
openFPGALoader --version     # v1.1.1
openFPGALoader --scan-usb    # lists 0x33aa:0x0120 (GWU2X) with a Chromatic connected
```

## Run
Start the app **from Terminal** in the window where the PATH above is set. A
Finder double-click does not inherit it, so firmware installs would not find
the tools.
```bash
<repo>/flutter/build/macos/Build/Products/Release/chromatic_pc_backup.app/Contents/MacOS/chromatic_pc_backup
```
The app must not be sandboxed (`com.apple.security.app-sandbox` is `false` in
`flutter/macos/Runner/Release.entitlements`) so it can run the tools.

## macOS notes
- macOS lists each USB serial device as both `/dev/cu.*` and `/dev/tty.*`.
  Auto-detect needs the fix in PR #4.
- Turn on **SYSTEM → C. MAGICIAN** on the handheld, or backup commands time out.
- The FPGA may report version `0.0` right after an install. Power-cycle the
  handheld.
- `esptool` can occasionally fail the first connect (`No serial data received`).
  Retry once.
- `cargo test -p chromatic-backup-core` has 14 failing tests on macOS
  (pty setup, "Not a typewriter"). The same 14 fail on upstream `main`.
EOF

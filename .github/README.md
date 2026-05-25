# MDM Patcher — macOS Build Guide

<div align="center">

**macOS command-line build notes for a libimobiledevice-based recovery utility**

[Overview](#overview) • [Legal--ethical-notice](#legal--ethical-notice) • [Requirements](#requirements) • [Build-on-macos](#build-on-macos) • [Required-runtime-files](#required-runtime-files) • [Troubleshooting](#troubleshooting)

</div>

---

## Legal & Ethical Notice

This project must only be used for lawful, authorised work on devices you personally own or have explicit written permission to service.

Do **not** use this on corporate, school, leased, financed, or otherwise managed devices unless the organisation that owns or manages the device has authorised you to do so. For managed devices, the correct fix is normally removal from Apple Business Manager, Apple School Manager, or the organisation's MDM console.

This README is only for building the macOS command-line project and resolving local dependency/runtime-file issues.

---

## Overview

This repository builds a macOS command-line tool that uses the `libimobiledevice` stack and local project assets.

This macOS-focused README assumes you are using Homebrew and already have, or can install, the required libraries with `brew`.

---

## Requirements

### Hardware

- Mac running macOS
- Apple Silicon or Intel Mac
- USB cable suitable for the iPhone/iPad
- iPhone/iPad that you are authorised to service

### Software

Install Xcode Command Line Tools:

```bash
xcode-select --install
```

Install Homebrew dependencies:

```bash
brew install \
  pkg-config \
  libplist \
  libimobiledevice \
  libimobiledevice-glue \
  libusbmuxd \
  libirecovery \
  openssl@3 \
  libzip \
  readline
```

---

## Build on macOS

### 1. Check Homebrew paths

On Apple Silicon Macs, Homebrew is usually installed under:

```bash
/opt/homebrew
```

On Intel Macs, it is usually:

```bash
/usr/local
```

You can confirm with:

```bash
brew --prefix
```

For Apple Silicon, export these paths before compiling:

```bash
export PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig:/opt/homebrew/opt/openssl@3/lib/pkgconfig:/opt/homebrew/opt/libzip/lib/pkgconfig:$PKG_CONFIG_PATH"
export CPPFLAGS="-I/opt/homebrew/include -I/opt/homebrew/opt/openssl@3/include -I/opt/homebrew/opt/readline/include"
export LDFLAGS="-L/opt/homebrew/lib -L/opt/homebrew/opt/openssl@3/lib -L/opt/homebrew/opt/readline/lib"
```

For Intel Homebrew, use:

```bash
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:/usr/local/opt/openssl@3/lib/pkgconfig:/usr/local/opt/libzip/lib/pkgconfig:$PKG_CONFIG_PATH"
export CPPFLAGS="-I/usr/local/include -I/usr/local/opt/openssl@3/include -I/usr/local/opt/readline/include"
export LDFLAGS="-L/usr/local/lib -L/usr/local/opt/openssl@3/lib -L/usr/local/opt/readline/lib"
```

---

### 2. Verify dependencies

Run:

```bash
pkg-config --modversion libimobiledevice-1.0
pkg-config --modversion libimobiledevice-glue-1.0
pkg-config --modversion libplist-2.0
pkg-config --modversion libirecovery-1.0
pkg-config --libs openssl libzip
```

Check device detection:

```bash
idevice_id -l
```

If your device is connected and trusted, this should print the device UDID.

---

### 3. Compile

From the project directory:

```bash
gcc -o mdm_patch \
    main.c \
    patch_logic.c \
    idevicebackup2.c \
    libidevicefunctions.c \
    utils.c \
    -I. \
    -D_GNU_SOURCE \
    -Wno-deprecated-declarations \
    $(pkg-config --cflags --libs \
      libimobiledevice-1.0 \
      libimobiledevice-glue-1.0 \
      libplist-2.0 \
      libirecovery-1.0 \
      openssl \
      libzip) \
    -L"$(brew --prefix readline)/lib" \
    -I"$(brew --prefix readline)/include" \
    -lreadline \
    -lm
```

Make it executable:

```bash
chmod +x mdm_patch
```

Confirm the binary exists:

```bash
ls -lah mdm_patch
```

---

## Required Runtime Files

The binary expects these files to be in the same directory you run it from:

```bash
extension1.pdf
extension2.pdf
libiMobileeDevice.dylib
```

Check with:

```bash
ls -lah extension1.pdf extension2.pdf libiMobileeDevice.dylib
```

Important: `libiMobileeDevice.dylib` is a project runtime asset expected by this code. It is **not** the same thing as Homebrew's normal `libimobiledevice-1.0.dylib`.

Homebrew's real dynamic library will usually look like:

```bash
/opt/homebrew/lib/libimobiledevice-1.0.dylib
```

Do not rename or symlink the Homebrew library to `libiMobileeDevice.dylib` unless you have checked the source and confirmed that is really what the program expects. In this project, the file appears to be opened at runtime as a local asset.

You can confirm where it is referenced with:

```bash
grep -R "libiMobileeDevice.dylib" -n .
```

---

## Running

From the folder containing the compiled binary and the required runtime files:

```bash
./mdm_patch
```

If the program reports a missing runtime file, check you are running it from the correct folder:

```bash
pwd
ls -lah
```

---

## Troubleshooting

### `fatal error: 'libirecovery.h' file not found`

Install `libirecovery`:

```bash
brew install libirecovery
```

Then verify:

```bash
pkg-config --modversion libirecovery-1.0
pkg-config --cflags --libs libirecovery-1.0
```

If needed, check the header exists:

```bash
find "$(brew --prefix)" -name libirecovery.h
```

---

### `Package ... was not found in the pkg-config search path`

Set `PKG_CONFIG_PATH`.

Apple Silicon:

```bash
export PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig:/opt/homebrew/opt/openssl@3/lib/pkgconfig:/opt/homebrew/opt/libzip/lib/pkgconfig:$PKG_CONFIG_PATH"
```

Intel:

```bash
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:/usr/local/opt/openssl@3/lib/pkgconfig:/usr/local/opt/libzip/lib/pkgconfig:$PKG_CONFIG_PATH"
```

Then retry:

```bash
pkg-config --list-all | grep -E "libimobiledevice|libplist|libirecovery|libzip|openssl"
```

---

### `plist_new_date is deprecated`

This is only a warning with newer Homebrew `libplist`.

Old code:

```c
plist_new_date(time(NULL) - MAC_EPOCH, 0)
```

New code for `libplist 2.7.0`:

```c
plist_new_unix_date(time(NULL))
```

For file modification time:

```c
plist_new_unix_date(st.st_mtime)
```

If you do not want to edit the source, suppress the warning during compile:

```bash
-Wno-deprecated-declarations
```

---

### `Error: Could not open file libiMobileeDevice.dylib`

The program is running but cannot find the expected runtime asset.

Check the current directory:

```bash
pwd
ls -lah
```

Find the file elsewhere:

```bash
find ~ -name "libiMobileeDevice.dylib" 2>/dev/null
```

Copy it into the same directory as the binary:

```bash
cp /path/to/libiMobileeDevice.dylib .
```

Then retry:

```bash
./mdm_patch
```

---

### `Error: Could not connect to device`

Check the device is visible to `libimobiledevice`:

```bash
idevice_id -l
ideviceinfo
```

Try:

```bash
brew services restart usbmuxd
```

Or unplug/replug the device and trust the Mac on the device when prompted.

---

### OpenSSL linking errors

Check what `pkg-config` returns:

```bash
pkg-config --cflags --libs openssl
```

For Apple Silicon, ensure OpenSSL paths are present:

```bash
export CPPFLAGS="-I/opt/homebrew/opt/openssl@3/include $CPPFLAGS"
export LDFLAGS="-L/opt/homebrew/opt/openssl@3/lib $LDFLAGS"
export PKG_CONFIG_PATH="/opt/homebrew/opt/openssl@3/lib/pkgconfig:$PKG_CONFIG_PATH"
```

---

## Project Layout

```text
mdmpatcher/
├── main.c
├── patch_logic.c
├── patch_logic.h
├── idevicebackup2.c
├── idevicebackup2.h
├── libidevicefunctions.c
├── libidevicefunctions.h
├── utils.c
├── utils.h
├── endianness.h
├── extension1.pdf
├── extension2.pdf
└── libiMobileeDevice.dylib
```

---

## Notes

- `sudo ldconfig` is Linux-only and should not be used on macOS.
- Do not build the whole `libimobiledevice` stack manually unless you specifically need unreleased upstream changes.
- Prefer Homebrew packages first.
- Keep the runtime files beside the compiled binary when testing.
- If a build error appears, fix the first `fatal error` or linker error first. Deprecation warnings are usually not the reason the build failed.

---

## Credits

- `libimobiledevice` project and contributors
- `libplist`, `libusbmuxd`, `libirecovery`, and related open-source maintainers
- Original project authors and contributors

---

## License

Use and distribute only in accordance with the licence terms of this repository and its dependencies. You are responsible for complying with applicable laws and any device ownership or management agreements.

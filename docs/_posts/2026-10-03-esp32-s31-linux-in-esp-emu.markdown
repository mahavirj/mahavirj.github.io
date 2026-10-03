---
layout: post
title:  "Booting ESP32-S31 Linux in an emulator"
date:   2026-10-03 21:00:00 +0530
categories: linux esp32s31 buildroot
---

Espressif's ESP32-S31 can run Linux, and the full stack — SPL, OpenSBI, U-Boot,
Linux 6.18 and a Buildroot root filesystem — is public on GitHub. You don't need
a board to try it: build the flash image with Buildroot and boot it in the
[ESP32 series emulator](https://github.com/espressif/esp-emulator). Here is the
short path.

## What you need

- An x86_64 or AArch64 Linux host (the prebuilt toolchain covers these two)
- The usual [Buildroot host packages](https://buildroot.org/downloads/manual/manual.html#requirement-mandatory)
- [`esptool`](https://github.com/espressif/esptool) with ESP32-S31 support (v5.3.0 used here)
- [`esp-emu`](https://github.com/espressif/esp-emulator) (v0.45.0 used here)

Install `esp-emu` with the installer from the
[esp-emulator](https://github.com/espressif/esp-emulator) repository. It puts the
binary in `~/.local/bin/esp-emu`:

```sh
curl -fsSL https://raw.githubusercontent.com/espressif/esp-emulator/main/install.sh | sh
esp-emu --version
```

Run `esp-emu update` later to get newer releases. The
[esp-emulator README](https://github.com/espressif/esp-emulator#readme) documents
every option. ESP32-S31 support in the emulator is still in early bring-up, so
use the latest release.

## 1. Get the sources

```sh
mkdir s31-linux && cd s31-linux

git clone --branch 2025.02 --depth 1 \
  https://gitlab.com/buildroot.org/buildroot.git buildroot

git clone https://github.com/espressif/esp-buildroot-external.git
```

Buildroot doesn't have to be a git clone. The official
[`buildroot-2025.02.tar.gz`](https://buildroot.org/downloads/buildroot-2025.02.tar.gz)
release tarball, unpacked as `buildroot`, works the same way.

Buildroot fetches everything else itself:

| Component | Source |
|-----------|--------|
| OpenSBI   | [`espressif/opensbi`](https://github.com/espressif/opensbi/tree/integration/v1.6-esp32s31) `integration/v1.6-esp32s31` |
| U-Boot    | [`espressif/u-boot`](https://github.com/espressif/u-boot/tree/integration/v2024.07-esp32s31) `integration/v2024.07-esp32s31` |
| Linux     | [`espressif/linux`](https://github.com/espressif/linux/tree/integration/v6.18-esp32s31) `integration/v6.18-esp32s31` |
| BSP tools | [`espressif/esp-linux-bsp`](https://github.com/espressif/esp-linux-bsp/tree/integration/v1.0-esp32s31) `integration/v1.0-esp32s31` |
| Toolchain | prebuilt `riscv64-esp-linux-musl` GCC 14.1.1 from `dl.espressif.com` |

## 2. Host compiler fix (GCC 15 only)

Stock Buildroot 2025.02 fails to build its host tools (`host-m4` first) with
GCC 15, which defaults to C23. If `gcc --version` says 15, apply this to the
`buildroot` tree:

```diff
--- a/package/m4/m4.mk
+++ b/package/m4/m4.mk
+HOST_M4_CONF_ENV = CFLAGS="$(HOST_CFLAGS) -std=gnu17"
--- a/package/bison/bison.mk
+++ b/package/bison/bison.mk
-HOST_BISON_CONF_ENV = ac_cv_libtextstyle=no
+HOST_BISON_CONF_ENV = ac_cv_libtextstyle=no CFLAGS="$(HOST_CFLAGS) -std=gnu17"
--- a/package/bash/bash.mk
+++ b/package/bash/bash.mk
+BASH_CONF_ENV += CFLAGS_FOR_BUILD="$(HOST_CFLAGS) -std=gnu17"
```

With CMake 4.x you also need `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` in
`HOST_LZO_CONF_OPTS` (`package/lzo/lzo.mk`), and a newer `host-dtc` may need
`-Wno-error` in its `EXTRA_CFLAGS`.

## 3. Configure and build

```sh
make -C buildroot \
  BR2_EXTERNAL=$PWD/esp-buildroot-external \
  O=$PWD/output \
  espressif_esp32s31_function_core_board_nor_defconfig

ESP_ESPTOOL=$(command -v esptool) \
  make -C buildroot O=$PWD/output -j"$(nproc)"
```

The result is a single 16 MiB NOR image:

```
output/images/s31_full_flash.bin
```

It holds the SPL, OpenSBI, U-Boot, the kernel (`xipImage`), the device tree and a
cramfs root filesystem, all at their flash offsets. You can boot it in the
emulator, or write it to a real board at `0x0` as described in the
[esp-buildroot-external README](https://github.com/espressif/esp-buildroot-external#deploy).

## 4. Boot it in esp-emu

```sh
esp-emu --chip esp32s31 --firmware output/images/s31_full_flash.bin --net user
```

The emulator runs the real ESP32-S31 mask ROM, which loads the image from the
emulated flash just as hardware would. The console is attached to your
terminal. A good boot looks like this (trimmed):

```
ESP-ROM:esp32s31-20251218
U-Boot SPL 2024.07
Trying to boot from NOR
OpenSBI: ESP32-S31 firmware active
U-Boot 2024.07
Starting kernel ...
[    0.000000] Linux version 6.18.0 (riscv64-esp-linux-musl-gcc 14.1.1) ...
[    0.000000] Machine model: Espressif ESP32-S31 (minimal)
[    0.906154] VFS: Mounted root (cramfs filesystem) readonly on device 31:0.
[    0.907937] Run /sbin/init as init process

~ # uname -a
Linux esp32s31 6.18.0 #1 ... riscv32 GNU/Linux
~ # free
              total        used        free      shared  buff/cache   available
Mem:          14776        1584       12632           0         560       12184
```

The board model has 16 MiB of PSRAM, the emulator's default for ESP32-S31, and
the root filesystem runs in place from flash, so about 12 MiB is left free
after boot.

### Scripted runs

`esp-emu` can type commands into the console and exit on a match, which makes
a handy smoke test, for example in CI:

```sh
esp-emu --chip esp32s31 --firmware output/images/s31_full_flash.bin \
  --timeout 300s \
  --inject-on '~ # ' --inject 'uname -a; echo BOOT_$((1+1))\n' \
  --exit-on 'BOOT_2'
```

It exits with 0 once the shell answers. Writing the marker as `$((1+1))` keeps
the echoed command itself from matching `--exit-on`.

## Things you will notice

- **No login prompt and no services.** The board's `inittab` mounts the
  filesystems and drops you straight into `/bin/sh`. `/etc/init.d` scripts do
  not run.
- **`/` is read-only.** Use `/tmp` or `/run` (tmpfs) for scratch files.
- **`ERROR: reserving fdt memory region failed (addr=2f030000 ...)`** in U-Boot
  is harmless. The kernel's DMA pool sits in the internal RAM that U-Boot
  itself runs from, and U-Boot refuses the overlapping reservation. The kernel
  and DTB are loaded into PSRAM, so nothing actually collides.

## Links

- [espressif/esp-buildroot-external](https://github.com/espressif/esp-buildroot-external) — Buildroot external tree for ESP32-S31
- [espressif/esp-emulator](https://github.com/espressif/esp-emulator) — `esp-emu` installer, releases and documentation
- [espressif/linux](https://github.com/espressif/linux), [espressif/u-boot](https://github.com/espressif/u-boot), [espressif/opensbi](https://github.com/espressif/opensbi), [espressif/esp-linux-bsp](https://github.com/espressif/esp-linux-bsp)

The ESP32-S31 Linux branch is a developer preview, so expect changes. Report
problems with the build on
[esp-buildroot-external](https://github.com/espressif/esp-buildroot-external/issues)
and problems with the emulator on
[esp-emulator](https://github.com/espressif/esp-emulator/issues).

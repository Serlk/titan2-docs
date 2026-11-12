<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Building mainline Linux for the Unihertz Titan 2

[Русская версия](BUILD-LINUX.ru.md) · Installing: [INSTALL-LINUX.md](INSTALL-LINUX.md) ·
What works: [STATUS.md](STATUS.md)

This page builds everything the installation needs, on a PC, from source. Nothing here touches
the phone. The commands run in the root of a `titan2-tools` checkout unless a step says
otherwise; every script prints its options when you read its header (`sed -n 1,40p <script>`).

## 1. The pieces

| repository | what it has |
|---|---|
| [titan2-linux](https://github.com/Serlk/titan2-linux), branch `titan2` | the kernel (mainline 7.2 based) and the device trees of the phone |
| [titan2-tools](https://github.com/Serlk/titan2-tools) | build scripts, the initramfs, the Alpine root file system builder, services and user space tools (modem file/RPC servers, call audio daemon, ModemManager plugin, the vendor runtime used by the optional parts) |
| [titan2-boot](https://github.com/Serlk/titan2-boot) | the boot menu that runs after the stock bootloader (LK) from `boot_a` |
| [titan2-twrp](https://github.com/Serlk/titan2-twrp) | TWRP recovery for the Titan 2 (needed once, for the repartitioning) |
| [titan2-camera](https://github.com/Serlk/titan2-camera) | optional: camera drivers (kernel patch series) and libcamera patches |
| [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | optional: fingerprint (kernel patch series and user space); not usable yet |
| [titan2-gnss](https://github.com/Serlk/titan2-gnss) | optional: GNSS with A-GPS (kernel patch series and user space) |
| [titan2-android](https://github.com/Serlk/titan2-android) | crDroid 16 for the Titan 2, the Android side of the dual boot |
| [titan2-docs](https://github.com/Serlk/titan2-docs) | this documentation |

What you end up with:

| output | made by | goes to |
|---|---|---|
| `out/mainline-build/` (kernel `Image.gz`, DTBs, modules) | `scripts/build-mainline.sh` | used by the next two |
| `out/bootmenu-dualboot/boot.img` | `scripts/make-bootmenu.sh` | partition `boot_a` |
| `out/linux-root/linux.img` | `scripts/make-linux-root.sh` | partition `linux` (created during the installation) |
| `out/t2-repo/` (optional) | `scripts/build-mesa.sh`, `scripts/build-libcamera.sh`, `scripts/t2-repo.sh` | packages for the root file system |

## 2. Host requirements

- A Linux PC (x86_64) with **Docker** (the kernel build, the image tools and the package
  builds all run in containers). The package builds (Mesa, libcamera) run aarch64 in emulation
  and need host kernel 6.7 or newer.
- `git`, `ssh`, `adb` and `fastboot` (Android platform-tools), `img2simg` (android-tools, makes
  the sparse `linux.img`), `e2fsprogs` (`e2fsck` for the root image, `debugfs` for the factory
  files), and for the optional factory-file extraction `lpunpack` (android-tools) and
  `erofs-utils` (`fsck.erofs`).
- Disk space: about **30 GB** for the kernel (toolchain 3.5 GB, sources, build tree about
  8 GB) and the root image (about 4 GB); about **60 GB** with the Mesa and libcamera builds.
- Time: the kernel takes 20-60 minutes (`JOBS=4` by default); Mesa takes hours in emulation.

## 3. Sources

```sh
git clone https://github.com/Serlk/titan2-tools
cd titan2-tools
git clone -b titan2 https://github.com/Serlk/titan2-linux linux       # the kernel tree (SRC=linux)
git clone https://github.com/Serlk/titan2-boot titan2-boot           # the boot menu
# AOSP mkbootimg (Apache-2.0, not included in titan2-tools)
git clone https://android.googlesource.com/platform/system/tools/mkbootimg tools/mkbootimg
```

`--depth 1` is fine for the kernel clone if you only build it. Use the heads of `titan2-linux`,
`titan2-boot` and `titan2-tools` from the same date: the boot menu hands memory and firmware
information over to the kernel, and an old menu with a new kernel can leave Wi-Fi without its
memory pool.

## 4. Toolchain and build container

The kernel does not build with clang 17; it needs the AOSP prebuilt **clang r574158**
(clang 21) in `toolchains/clang-r574158`. It is the directory `clang-r574158` of
<https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86>; gitiles can give
you that one directory as a tarball (the "tgz" link on the directory page, a URL of the form
`.../linux-x86/+archive/refs/heads/<branch>/clang-r574158.tar.gz`):

```sh
mkdir -p toolchains/clang-r574158
tar -xzf clang-r574158.tar.gz -C toolchains/clang-r574158
toolchains/clang-r574158/bin/clang --version      # clang version 21...
docker build -t titan2-kbuild docker/             # the build container used by every script
make -C tools CC=$PWD/toolchains/clang-r574158/bin/clang     # small static helpers
```

## 5. The kernel

```sh
scripts/build-mainline.sh                  # -> out/mainline-build
```

It runs `make defconfig`, merges `configs/mainline-titan2.config`, and builds `Image.gz`,
every `mt6878-unihertz-titan2*.dtb` of the tree and the modules (installed stripped into
`out/mainline-build/modules/lib/modules/<version>`). Variables: `SRC=linux` (the kernel tree),
`O=out/mainline-build` (build directory), `JOBS=4`, `MODULES=0` (kernel and DTBs only).

Device tree variants (one menu entry each):

| DTB | what it is |
|---|---|
| `mt6878-unihertz-titan2-all.dtb` (`all`) | **everything that works**: display (mediatek-drm), GPU, Wi-Fi, Bluetooth, modem, audio, charger, sensors. Use this one. |
| `mt6878-unihertz-titan2-all-fb.dtb` (`all-fb`) | the same, but the display stays on the bootloader's framebuffer (simplefb). Fallback when the screen stays black. |
| `mt6878-unihertz-titan2.dtb` (`full`) | the base: framebuffer display, keyboard, touch, buttons, storage, USB, PMIC, fuel gauge. No radio, no GPU. The safest entry. |
| `-all-iommu` | development: the display behind the IOMMU. Shows garbage, do not use. |
| `-drm`, `-gpu`, `-wifi`, `-modem`, `-audio`, `-pd`, `-dvfs`, `-idle`, `-jpeg`, `-charger-*` | single-subsystem test variants for development |

Do not edit the kernel tree while a build runs (the container builds whatever is on disk at
that moment), and give parallel builds their own `O=` directories.

### With the optional features

Camera, fingerprint and GNSS live in their own repositories. Each carries its kernel part as a
patch series against the `titan2` branch and a config fragment; the README of each repository
names the exact files. The steps are the same for all three, for example the camera:

```sh
git clone https://github.com/Serlk/titan2-camera
cd linux
git checkout -b titan2-local titan2
git am ../titan2-camera/<kernel patch directory>/*.patch      # path: see the titan2-camera README
cd ..
cp configs/mainline-titan2.config configs/mainline-titan2.config.orig
cat titan2-camera/<config fragment> >> configs/mainline-titan2.config
O=out/mainline-build scripts/build-mainline.sh
```

`scripts/build-mainline.sh` merges only `configs/mainline-titan2.config`, so the fragment is
appended to it (later lines win in the merge). Several features can be stacked: apply the series
one after another (camera, then GNSS, then fingerprint) and append each fragment. If a series
does not apply, your `titan2` checkout is not the one the series was made for: check out the base
commit named in that repository's README.

## 6. The initramfs

You do not have to build it separately: `make-bootmenu.sh`, `make-linux-root.sh` and
`linux-update.sh` call `tools/initramfs/build.sh` themselves. It downloads Alpine v3.24
aarch64 packages (busybox, dropbear, e2fsck, ...) and checks their signatures against the Alpine
key in `tools/initramfs/keys`. What it does on the phone: USB network (the phone is
172.16.42.1 and gives the PC an address by DHCP) and a serial console, SSH, finding and checking
the root (`t2.root=part:linux`), growing the `linux` file system on first boot, and making
every other UFS partition read-only with a one-way kernel lock (see
[DUALBOOT.md](DUALBOOT.md), "Protection").

**SSH keys:** the scripts take your public key from `AUTHORIZED_KEYS=<file>`, else
`tools/initramfs/authorized_keys`, else `~/.ssh/id_ed25519.pub`. Without any key, root has an
**empty password** (SSH is then reachable over the USB cable and, once Wi-Fi is up, over Wi-Fi
too). Always pass a key.

## 7. Packages: Mesa, libcamera (recommended / optional)

Alpine v3.24 has no Mesa that supports the phone's Mali-G615 (Valhall v11); without our build
the desktop renders on the CPU (llvmpipe). **Mesa 26.3.0** is the first release with the G615;
`scripts/build-mesa.sh` builds it from Alpine's APKBUILD plus a few fixes (panfrost for
OpenGL ES, panvk for Vulkan, llvmpipe/lavapipe as fallback):

```sh
scripts/build-mesa.sh build          # hours; -> out/mesa/packages/aports/aarch64
scripts/build-libcamera.sh build     # only for the camera; -> out/libcamera/packages/aports/aarch64
scripts/t2-repo.sh build             # one signed index over all of them -> out/t2-repo
scripts/t2-repo.sh list
```

The first package build creates your own signing key in `out/atl/build-keys/`
(`titan2-atl.rsa`, `.pub`); the root image trusts that public key. Keep it if you want to
update the packages on the phone later. `build-mesa.sh status|clean` and
`build-libcamera.sh status|clean` show and remove the build trees.
`scripts/build-atl.sh` builds the Android Translation Layer (running some Android apps as
Linux programs); that is experimental and not needed for anything on this page.

## 8. The root file system (`linux.img`)

```sh
LOCAL_REPO=out/t2-repo AUTHORIZED_KEYS=~/.ssh/id_ed25519.pub \
EXTRA_PKGS="modemmanager modemmanager-openrc networkmanager-wwan polkit polkit-openrc \
    alsa-lib alsa-utils pipewire wireplumber pipewire-pulse" \
    scripts/make-linux-root.sh
```

Result: `out/linux-root/linux.img` (sparse, for `fastboot flash linux`) and `linux.raw` (the
plain ext4). The script builds an Alpine v3.24 aarch64 system (`scripts/netroot-serve.sh build`
under the hood: OpenRC, eudev, NetworkManager with Wi-Fi, dropbear, chrony, the Titan 2
services and configuration from `tools/netroot/`), installs the kernel modules of
`out/mainline-build`, and adds `/boot` (kernel, initramfs, all DTBs and
`/boot/extlinux/extlinux.conf` with one entry per DT variant, root `t2.root=part:linux`).
Package scripts that cannot run on x86 run on the phone at the first boot.

Variables:

| variable | meaning |
|---|---|
| `KO=out/mainline-build` | the kernel build to install |
| `SIZE=8G` | size of the ext4 before it is grown to the whole partition on the phone |
| `VARIANTS="all all-fb full"` | which DT variants get an entry (default: all built ones) |
| `GFX=1` (default) | graphics: niri, foot, alacritty, rofi, Xwayland bridge, weston/cage for tests, iio-sensor-proxy, BlueZ |
| `LOCAL_REPO=out/t2-repo` | your package repository; its Mesa (and with `CAMERA=1` libcamera) replace Alpine's |
| `CAMERA=1` | camera: libcamera, PipeWire + WirePlumber, portals, GNOME Snapshot, qcam (needs libcamera in `LOCAL_REPO` and a kernel with the camera series) |
| `EXTRA_PKGS="..."` | more Alpine packages; the example above adds what the modem, calls and audio need |
| `AUTHORIZED_KEYS=<file>` | SSH keys for root (see section 6) |
| `REUSE=1` | keep the already built Alpine image, only redo `/boot` |
| `OUT=out/linux-root` | output directory |

The optional parts that have their own repositories (GNSS, fingerprint) come with their own
packages and install instructions; see [INSTALL-LINUX.md](INSTALL-LINUX.md#12-optional-camera-gnss-fingerprint).

## 9. The boot menu (`boot_a`)

The menu image contains the **kernel of the Android you have installed**, so take it from your
phone (the backup of `boot_a` made before the installation, see INSTALL-LINUX.md step 2). The
file must be inside the titan2-tools directory.

```sh
ANDROID_BOOT=backup/<date>-pre-linux/boot_a.img OUT=out/bootmenu-dualboot \
    UFS=1 LINUX_ROOT=part:linux VARIANTS=full DEFAULT=android \
    scripts/make-bootmenu.sh
```

Result: `out/bootmenu-dualboot/boot.img`. The menu has: Android, `Linux <version> (boot_a)` (a
fallback kernel inside `boot_a` with the root on the `linux` partition), "Linux partition" (the
entries of `/boot/extlinux/extlinux.conf` on the `linux` partition, read by the menu's own UFS
reader), Fastboot, Recovery, Reboot.

| variable | meaning |
|---|---|
| `ANDROID_BOOT=<boot.img>` | required: the Android boot image whose kernel goes into the menu. Never a `boot_a` that already holds the menu. |
| `UFS=1` | adds "Linux partition"; the UFS is read only when that entry is chosen (or is the default) |
| `LINUX_ROOT=part:linux` | root of the fallback Linux entries |
| `VARIANTS=full` | DT variants of the fallback entries (`full` is the safe one) |
| `DEFAULT=android\|linux\|linux-<variant>\|ufs`, `TIMEOUT=5` | default entry and countdown in seconds |
| `NO_LINUX=1` | a menu with Android only |
| `CMDLINE` | kernel command line of the fallback entries |
| `OUT=out/bootmenu` | output directory |

Recovery (TWRP), power-off charging and the alarm boot never show the menu and always start
Android's kernel, so they do not depend on anything Linux-side.

## 10. Phone-side programs that are not packaged yet

The modem and call stack consists of small C programs in titan2-tools. They are not apk
packages yet; the simplest way is to build them on the phone (Alpine has the compilers):

| program | directory | role |
|---|---|---|
| `t2-ccci-fsd` | `tools/ccci-fsd` | answers the modem's NVRAM file requests from a sandbox copy (never from a flash partition) |
| `t2-ccci-rpcd` | `tools/ccci-rpcd` | answers the modem's RPC requests (reads the modem image read-only) |
| `t2-speechd` | `tools/speechd` | call audio: AP audio path and the modem's speech engine |
| ModemManager plugin `mtk-ccci` | `tools/mm-titan2` | calls, SMS, mobile data, IMS (VoLTE) through ModemManager |
| `t2-powerd` (optional) | `tools/powerd` | experimental sleep manager |

Each directory has a README with build, test and run instructions; `make test` runs the host
tests. The installation of the modem stack on the phone is in
[INSTALL-LINUX.md, section 10](INSTALL-LINUX.md#10-mobile-network-calls-and-sms).

## 11. Tests without the phone

```sh
scripts/test-gpt-carve.sh           # the repartitioning on an image with the Titan 2 GPT layout
scripts/test-bootmenu-qemu.sh       # the boot menu in qemu (Android, Linux, recovery, disk entries)
tools/initramfs/qemu/diskroot.sh    # root on a partition, the read-only lock, write attempts as root
```

The boot menu's own unit tests are described in the titan2-boot README (`make test`, `make qemu`).

Next: [INSTALL-LINUX.md](INSTALL-LINUX.md).

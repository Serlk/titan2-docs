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
| `out/t2-repo/` (optional, needed for the GPU) | `scripts/t2-repo.sh` (our aports), `scripts/build-mesa.sh` (Mesa 26.3), titan2-camera's `scripts/build-libcamera.sh` | packages for the root file system |

## 2. Host requirements

- A Linux PC (x86_64) with **Docker** (the kernel build, the image tools and the package
  builds all run in containers). The package builds run aarch64 in emulation: our aports and
  Mesa in `alpine` arm64 containers, which need qemu-user binfmt support for aarch64 (e.g. the
  `qemu-user-static` package; an arm64 PC needs nothing extra), libcamera with host kernel 6.7
  or newer.
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
memory pool. **Kernels from 2026-11-12 on need titan2-boot from 2026-11-12 or newer** (the commit
"take the reserved-memory rules from the mainline DTB"): their Wi-Fi device tree asks the menu to
turn LK's Wi-Fi pool into a restricted DMA pool (`titan2,lk-refs`), which older menus ignore, and
the Wi-Fi driver then refuses the pool. The same menu version scrolls the "Linux partition" list
and shows up to 24 entries (older: 12).

The optional parts (section 5) are separate repositories; clone the ones you want next to
`titan2-tools` (the paths below assume `../titan2-camera`, `../titan2-fingerprint`,
`../titan2-gnss`):

```sh
cd ..
git clone https://github.com/Serlk/titan2-camera
git clone https://github.com/Serlk/titan2-fingerprint
git clone https://github.com/Serlk/titan2-gnss
cd titan2-tools
```

## 4. Toolchain and build container

The kernel does not build with clang 17; it needs **clang 21**. The simplest is the official LLVM
release on GitHub ([llvmorg-21.1.8](https://github.com/llvm/llvm-project/releases/tag/llvmorg-21.1.8), file `LLVM-21.1.8-Linux-X64.tar.xz`, about 2 GB); the kernel
builds with it without errors. `CLANG_DIR` selects the compiler (default `toolchains/clang-r574158`):

```sh
mkdir -p toolchains
curl -L -o toolchains/LLVM-21.1.8-Linux-X64.tar.xz \
  https://github.com/llvm/llvm-project/releases/download/llvmorg-21.1.8/LLVM-21.1.8-Linux-X64.tar.xz
echo "b3b7f2801d15d50736acea3c73982994d025b01c2f035b91ae3b49d1b575732b  toolchains/LLVM-21.1.8-Linux-X64.tar.xz" | sha256sum -c -
mkdir -p toolchains/llvm-21.1.8
tar -xJf toolchains/LLVM-21.1.8-Linux-X64.tar.xz -C toolchains/llvm-21.1.8 --strip-components=1
toolchains/llvm-21.1.8/bin/clang --version        # clang version 21.1.8
export CLANG_DIR=$PWD/toolchains/llvm-21.1.8      # for scripts/build-mainline.sh
docker build -t titan2-kbuild docker/             # the build container used by every script
make -C tools CC=$CLANG_DIR/bin/clang             # small static helpers
```

The GitHub LLVM release uses the system `libxml2.so.2`: the container has it (docker/Dockerfile); for
`make -C tools` on the PC itself install the `libxml2` package. If you already have the AOSP prebuilt **clang r574158** (what we build with; it is self-contained and
needs no `libxml2`), put it into `toolchains/clang-r574158`, then `CLANG_DIR` is not needed. That version
is no longer on the `main` branch of AOSP `prebuilts/clang`, so download the GitHub LLVM release instead.

## 5. The kernel

```sh
scripts/build-mainline.sh                  # -> out/mainline-build
```

It runs `make defconfig`, merges `configs/mainline-titan2.config` (and the `FRAGMENTS`, below),
and builds `Image.gz`, every `mt6878-unihertz-titan2*.dtb` of the tree and the modules
(installed stripped into `out/mainline-build/modules/lib/modules/<version>`). Variables:
`SRC=linux` (the kernel tree, inside titan2-tools), `O=out/mainline-build` (build directory),
`JOBS=4`, `MODULES=0` (kernel and DTBs only), `FRAGMENTS="<file> ..."` (more config fragments,
merged after the base config in the order given; paths relative to the current directory, may be
outside titan2-tools).

Device tree variants (one menu entry each):

| DTB | what it is |
|---|---|
| `mt6878-unihertz-titan2-all.dtb` (`all`) | **everything that works**: display (mediatek-drm), GPU, Wi-Fi, Bluetooth, modem, audio, charger, sensors. Use this one. |
| `mt6878-unihertz-titan2-all-fb.dtb` (`all-fb`) | the same, but the display stays on the bootloader's framebuffer (simplefb). Fallback when the screen stays black. |
| `mt6878-unihertz-titan2.dtb` (`full`) | the base: framebuffer display, keyboard, touch, buttons, storage, USB, PMIC, fuel gauge. No radio, no GPU. The safest entry. |
| `-all-iommu` | development, only with the camera series: the display behind the IOMMU. Shows garbage, do not use. |
| `-drm`, `-gpu`, `-wifi`, `-modem`, `-audio`, `-pd`, `-dvfs`, `-idle`, `-jpeg`, `-charger-*` | single-subsystem test variants for development |

Do not edit the kernel tree while a build runs (the container builds whatever is on disk at
that moment), and give parallel builds their own `O=` directories.

### With the optional features

Camera, fingerprint and GNSS are not in titan2-linux. Each lives in its own repository, which
carries its kernel part as a patch series against the `titan2` branch (`kernel/patches/*.patch`,
made with `git format-patch` on `titan2` commit `50a22e5003dd`) and a config fragment
(`kernel/<name>.config`); its `kernel/README.md` has the details. Apply the series on a local
branch of your `titan2` checkout and pass the fragments to the build:

```sh
cd linux
git checkout -b titan2-local titan2           # or the base commit: git checkout -b titan2-local 50a22e5003dd
git am ../../titan2-camera/kernel/patches/*.patch
git am ../../titan2-fingerprint/kernel/patches/*.patch
git am ../../titan2-gnss/kernel/patches/*.patch
cd ..
FRAGMENTS="../titan2-camera/kernel/titan2-camera.config \
    ../titan2-fingerprint/kernel/titan2-fingerprint.config \
    ../titan2-gnss/kernel/titan2-gnss.config" \
    scripts/build-mainline.sh
```

Take only the lines of the parts you want; the three series touch different hunks and apply one
after another (tested in the order camera, fingerprint, gnss). A fragment without its series is
harmless but has no effect (its options have no Kconfig entry and are dropped). If `git am`
stops on a newer `titan2`, run `git am --abort` and start the local branch from the base commit
named above. The camera series adds the cameras to the `all` DT; the fingerprint and GNSS series
add their nodes to the base board and the Wi-Fi DT.

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

## 7. Packages: your own apk repository (optional)

Packages that are not in Alpine v3.24 are built into your own signed apk repository
`out/t2-repo`, which the root image build takes as `LOCAL_REPO`:

```sh
scripts/t2-repo.sh keygen                  # once: out/t2-repo-keys/t2-repo.rsa(.pub); keep it
make -C ../titan2-fingerprint dist; make -C ../titan2-gnss dist     # their source archives
scripts/t2-repo.sh aport tools/aports/titan2-vendor-runtime \
    ../titan2-fingerprint/aports/titan2-fingerprint \
    ../titan2-gnss/aports/titan2-gnss ../titan2-gnss/aports/titan2-epo   # -> out/t2-packages
scripts/build-mesa.sh build                # Mesa 26.3 for the GPU (hours) -> out/t2-packages
scripts/t2-repo.sh build                   # one signed index -> out/t2-repo
scripts/t2-repo.sh list
```

`aport` builds aarch64 packages in an `alpine` arm64 container, so the PC needs qemu-user binfmt
support for aarch64 (e.g. the `qemu-user-static` package of your distribution). The image
trusts the public key `out/t2-repo/t2-repo.rsa.pub`; keep the key pair if you want to update
the packages on the phone later.

- **Camera**: titan2-camera's `scripts/build-libcamera.sh build` builds the patched libcamera
  (hours in emulation, host kernel 6.7 or newer). Sign it with the same key and put the output
  inside titan2-tools so that `t2-repo.sh build` can index it:

  ```sh
  (cd ../titan2-camera && TITAN2_TOOLS=../titan2-tools KEYS=../titan2-tools/out/t2-repo-keys \
      OUT=../titan2-tools/out/libcamera scripts/build-libcamera.sh build)
  EXTRA_APK_DIRS=out/libcamera/packages/aports/aarch64 scripts/t2-repo.sh build
  ```

- **Fingerprint and GNSS** also need `bionic_translation` with the patches in
  `tools/titan2-vendor-runtime/patches/bionic_translation` (see that README); build it from
  Alpine's aport with these patches and add the `.apk` files with `EXTRA_APK_DIRS=` as above.
- **Mesa**: Alpine v3.24 has no Mesa that supports the phone's Mali-G615 (Valhall v11). Mesa
  26.3.0 is the first release with the G615. `scripts/build-mesa.sh build` builds it from
  Alpine's aport (pinned commit, Mesa archive checked against its sha512) with
  `tools/mesa/APKBUILD.diff` (panfrost/panvk plus the software fallback only) and
  `patches/mesa/*.patch`, in an `alpine:3.24` arm64 container: several hours under emulation and
  about 25 GB of disk. The packages are signed with your `out/t2-repo-keys/t2-repo.rsa` (created
  if it does not exist) and copied to `out/t2-packages/aarch64`, so `t2-repo.sh build` indexes
  them; the image build then upgrades the Mesa packages from `LOCAL_REPO`. Without this step the
  desktop renders on the CPU (llvmpipe). `scripts/build-mesa.sh fetch` only downloads and
  prepares the sources, to check them first.

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
| `VARIANTS="all all-fb full"` | which DT variants get an entry, in menu order (default: `all`, `all-fb`, `full`, then the other built ones) |
| `DEFAULT=all` | the default entry of `/boot/extlinux/extlinux.conf` (labels are the variant names) |
| `GFX=1` (default) | graphics: niri, foot, alacritty, rofi, Xwayland bridge, weston/cage for tests, iio-sensor-proxy, BlueZ |
| `LOCAL_REPO=out/t2-repo` | your package repository; its Mesa (and with `CAMERA=1` libcamera) replace Alpine's |
| `CAMERA=1 CAMERA_DIR=../titan2-camera` | camera: libcamera, PipeWire + WirePlumber, portals, GNOME Snapshot, qcam (needs our libcamera in `LOCAL_REPO` and a kernel with the camera series) |
| `FP=1 FP_DIR=../titan2-fingerprint` | fingerprint packages (needs them, `titan2-vendor-runtime` and `bionic_translation` in `LOCAL_REPO`) |
| `GNSS=1 GNSS_DIR=../titan2-gnss` | GNSS packages, GeoClue (same requirements as `FP=1`) |
| `EXTRA_PKGS="..."` | more Alpine packages; the example above adds what the modem, calls and audio need |
| `AUTHORIZED_KEYS=<file>` | SSH keys for root (see section 6) |
| `REUSE=1` | keep the already built Alpine image, only redo `/boot` |
| `OUT=out/linux-root` | output directory |

`CAMERA`, `FP` and `GNSS` hand each repository to the image builder: its `netroot/packages` are
installed, its `netroot/install.d/*.sh` set up its services. Without `*_DIR` the builder looks in
`external/titan2-<name>` inside titan2-tools. All three kernel and package parts together:

```sh
LOCAL_REPO=out/t2-repo AUTHORIZED_KEYS=~/.ssh/id_ed25519.pub \
CAMERA=1 CAMERA_DIR=../titan2-camera FP=1 FP_DIR=../titan2-fingerprint GNSS=1 GNSS_DIR=../titan2-gnss \
    scripts/make-linux-root.sh
```

What each part needs on the phone afterwards (files from your own phone, services): see
[INSTALL-LINUX.md](INSTALL-LINUX.md#12-optional-camera-gnss-fingerprint).

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

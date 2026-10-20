<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Booting mainline Linux through the stock LK: interface specification

This document describes what the stock Little Kernel (LK) bootloader of the
Unihertz Titan 2 (firmware V01.00.13 and V01.00.14) expects from the images it
boots and from the device tree blob (DTB) it hands to the kernel. It is written
as a behaviour specification: inputs, checks LK performs, what LK adds, and what
makes it stop. The bootloader itself is never modified; everything below is
satisfied by the images we build.

Both LK versions shipped on the phone behave the same way for everything
described here.

The requirements were found from LK's boot log, from the device trees it
passes to the factory kernel and to ours, from our boot attempts, and from an
analysis of the behaviour of the shipped LK binary. No LK code is reproduced
here.

Conventions:

- "LK stops" means LK hits an assertion or panic, prints it to its log and
  reboots. Each such reboot consumes one boot attempt of the active slot.
- "Optional" means LK logs a message and continues when the node or property is
  missing.
- The LK log of a failed boot survives in the `expdb` partition and can be read
  from Android afterwards (it is a plain text ring buffer). This is the main
  diagnostic channel when the kernel never starts; there is no accessible UART.

Contents:

1. Image set and header versions
2. Load addresses
3. Kernel image format
4. Kernel command line and bootconfig
5. Ramdisk assembly (vendor ramdisk fragments, init_boot)
6. DTB selection and dtbo overlay
7. Required DTB structure
8. Nodes required by the firmware loaders
9. What LK adds to the DTB
10. Verified boot (vbmeta)
11. A/B slots and retries
12. Getting out of a boot loop
13. Checklist

---

## 1. Image set and header versions

LK boots Android boot image header **version 4**, as stock does. With v4 three
partitions of the current slot are involved:

| partition | content we put there | requirement |
|---|---|---|
| `boot_<slot>` | compressed kernel + command line | magic `ANDROID!`, header v4 |
| `vendor_boot_<slot>` | vendor ramdisk (our initramfs) + DTB + vendor cmdline | magic `VNDRBOOT`, valid header |
| `init_boot_<slot>` | an **empty** cpio | magic `ANDROID!` must be present, otherwise LK stops |
| `dtbo_<slot>` | zeros (at least 4 KiB) | see section 6 |
| `vbmeta_<slot>` | stock vbmeta re-flashed with verification disabled | see section 10 |

With v4 the ramdisk field of `boot` is ignored. `init_boot` must exist and be a
valid image: if the stock `init_boot` is left in place, the stock first-stage
`/init` is unpacked on top of our initramfs, so it must be replaced by an image
with an empty cpio.

LK also contains a header v2 path (everything in `boot`, only the `ANDROID!`
magic checked) and v0/v1 paths, but these were not tried on this device.

No MediaTek signature (certificate chain) is needed on `boot`, `vendor_boot` or
`dtbo` on an unlocked device; image authentication depends on the security
configuration, not on the images. LK shows the "orange state" warning for about
five seconds.

## 2. Load addresses

The preloader reserves fixed memory blocks before LK runs. The image headers
must name exactly these addresses, otherwise LK stops:

| what | address | size of the block | notes |
|---|---|---|---|
| kernel | `0x40000000` | 0x7c80000 (124.5 MiB) | upper bound for the **decompressed** kernel including BSS (must end below `0x47c80000`) |
| DTB | `0x47c80000` | 4 MiB | LK places the DTB at the header's **`tags_addr`**, not at `dtb_addr` |
| ramdisk | `0x66f00000` | 64 MiB | all ramdisk pieces plus bootconfig (+0x400) must fit |

Equivalent mkbootimg arguments:

```
--header_version 4 --base 0 --pagesize 4096 \
--kernel_offset 0x40000000 --ramdisk_offset 0x66f00000 \
--tags_offset 0x47c80000 --dtb_offset 0x47c80000
```

The 4 MiB DTB block must hold the final tree including everything LK appends
(section 9); LK's own memory map property alone is about 22 KiB. Keep the DTB
well below 3.5 MiB.

## 3. Kernel image format

- The kernel must be **compressed**: gzip (`Image.gz`) or legacy LZ4
  (`lz4 -l`, 8 MiB blocks). An uncompressed `Image` makes LK panic because it
  finds no compression magic.
- LK decompresses the kernel to `0x40000000` and enters it through a secure
  monitor call to the ARM Trusted Firmware, with the DTB at `0x47c80000`.
- The kernel starts at **EL1** (the boot chain runs a hypervisor, GenieZone,
  above it); KVM is therefore not available. Confirmed in the kernel log of boot
  attempt 4: "All CPU(s) started at EL1".

## 4. Kernel command line and bootconfig

The final command line is assembled in this order:

1. `bootargs` from the DTB `/chosen` node;
2. LK's own additions, e.g. `root=/dev/ram` (from the AVB code path),
   `earlycon=uart8250,mmio32,0x11001000 console=ttyS0,921600n1`,
   `ramoops.mem_address=0x48090000 ramoops.mem_size=0xe0000 ...`, `bootopt=...`;
3. the command line of `boot`;
4. the command line of `vendor_boot`.

Consequences and limits:

- The total length is limited to **2047 bytes**; longer command lines make LK
  panic.
- All `androidboot.*` parameters are moved into a **bootconfig** block (limit
  1 KiB) appended to the end of the initrd, and `bootconfig` is added to the
  command line. The kernel needs `CONFIG_BOOT_CONFIG=y`.
- Because the last `console=` wins, a `console=tty0` placed in the DTB
  `bootargs` would be overridden by LK's `console=ttyS0`; put it in the `boot`
  command line instead. The same holds for `root=`: the `boot` command line
  comes after LK's `root=/dev/ram` and therefore takes effect.
- LK does not remove `console=` or `loglevel=` parameters; its key-replacement
  logic only applies to DTB `bootargs`.

### The command line offset quirk (header v3/v4)

LK copies the `boot` header into a buffer laid out like a **v0-v2** header and
reads the command line at offset **64** (the v0-v2 position). In a v3/v4 header
the command line starts at offset **44**. As a result the **first 20 bytes** of
the v4 `boot` command line are lost: in boot attempt 3,
`console=tty0 loglevel=7` arrived in the kernel as `l=7`.

Stock Android never notices because its `boot` command line is empty. The
`vendor_boot` command line is read correctly.

Workaround used by our image script: prefix the `boot` command line with 20
spaces, and check that what LK will read from offset 64 is the intended string.

### KASLR seed

LK fills `/chosen/kaslr-seed` from its hardware random number generator **only
if the property already exists**, and stops if its length is not exactly 8
bytes. Without the property the kernel logs "KASLR disabled due to lack of
seed". Provide `kaslr-seed = /bits/ 64 <0>;`.

## 5. Ramdisk assembly

With header v4 LK builds the initrd at `0x66f00000` as:

1. the vendor ramdisk fragment of type **PLATFORM** (in recovery mode also the
   fragment of type **RECOVERY**);
2. the `init_boot` ramdisk;
3. the bootconfig block.

Details:

- The bootconfig block is placed at `start + total size of all vendor ramdisk
  fragments + init_boot ramdisk size`, i.e. LK reserves room for the RECOVERY
  fragment even in a normal boot where it does not copy it. If such a fragment
  exists, a gap of stale memory remains between the last archive and
  bootconfig; the kernel then logs "Initramfs unpacking failed: invalid magic at
  start of compressed archive". This happens after all real archives and is
  harmless (bootconfig is located from the end of the initrd).
- **Recovery mode** (entered with Volume Up during boot) looks up a vendor
  ramdisk entry of type RECOVERY and stops if there is none. A `vendor_boot`
  with only a PLATFORM fragment therefore cannot be used through recovery; add
  an (even empty) fragment with `--ramdisk_type recovery` if that path is
  wanted. Normal boot is unaffected.
- `linux,initrd-start` / `linux,initrd-end` are written by LK as 4-byte (single
  cell) values.

## 6. DTB selection and dtbo overlay

### DTB

- With header v3/v4 the DTB comes from the dtb section of `vendor_boot`. An
  empty dtb section makes LK stop.
- If the section starts with a DT table header (magic `0xd7b7ab1e`), LK skips
  the table and uses the **first** blob, without matching board id or revision.
  A raw FDT without a table is accepted as is.
- LK copies the DTB into the 4 MiB block at `0x47c80000` and edits it there.
- LK configures its own drivers (display, PMIC, connectivity) from its own
  built-in copy of the stock tree, not from ours, so our DTB does not affect
  what LK does to the hardware before the kernel.

### dtbo

- LK reads `dtbo_<slot>` into a 1 MiB buffer.
- If the partition has neither a DT table header nor a MediaTek header, LK logs
  that no known header was found and **continues without an overlay**. A zeroed
  `dtbo` is therefore safe and is what we use.
- If a DT table is larger than 1 MiB, LK stops.
- An entry is chosen by index; if none matches, entry 0 is used. A table with
  zero entries is not safe (LK reads "entry 0" anyway).
- If an overlay is read but **fails to apply** (for example because the stock
  overlay's fixups do not resolve against a mainline tree), LK stops. The stock
  `dtbo` must never be left next to a mainline DTB.
- There is no "dconfig" overlay on this device, and errors there would not be
  fatal.

## 7. Required DTB structure

After the overlay step LK completes the DTB in a series of fixup steps. It
stops in this phase only when a step asserts, which
happens when a node it needs is missing or when a property write fails. Path
lookups use libfdt semantics, so `emicen` matches `emicen@10219000`.

### 7.1 Mandatory nodes and properties

| requirement | why |
|---|---|
| root `#address-cells = <2>; #size-cells = <2>;` | LK writes 16-byte `reg` values (2+2 cells) into `/memory` and the reserved-memory nodes and reads them back that way |
| `/chosen` | target of most fixup steps: boot tags, command line, initrd, security info, chip id, device info |
| `/chosen/bootargs` (any string, even empty, but the property must exist) | the command line finaliser and the bootconfig builder fail without it |
| `/memory` (`memory@40000000`, the first one counts) | LK replaces `reg` with the real DRAM size (12 GiB: `<0 0x40000000 3 0>`) and adds its memory map |
| `/reserved-memory` with `#address-cells = <2>; #size-cells = <2>; ranges;` | LK adds its carveouts here; it panics if the node is missing |
| a node named `emicen` (in `/soc` or at the root) | memory controller information |
| a node named `emiisu` (in `/soc` or at the root) | same |
| a node with `compatible = "mediatek, soc-dbg-error-flag"` (**with the space after the comma**) | the preloader passes debug error flag records; LK asserts if it cannot find a node for them |
| a node `/wifi` (or `/soc/wifi`), e.g. `wifi@18400000 { status = "disabled"; }` | LK can attach a Wi-Fi carveout to it as `memory-region`; strictly required only when the connectivity loader runs (section 8), harmless otherwise |
| a node with `compatible = "mediatek,scp-audio-mbox"` carrying `adsp-rsv-*` size properties with a non-zero sum | see 7.3; without it LK panics on a zero-sized allocation (boot attempt 2) |

Empty placeholder nodes are enough for `emicen`, `emiisu` and the debug-flag
node; preferably mark them `status = "disabled"`. LK only adds properties to
them (channel/rank counts, sizes, register offsets and so on).

### 7.2 Forbidden or constrained content

- **No static `reg` entries in `/reserved-memory` that overlap LK's carveouts.**
  Even an identical region counts as an overlap and makes LK stop. The stock
  tree only has dynamic nodes there (`size` + `alloc-ranges`). LK adds the ATF,
  TEE, modem, SCP, pstore and framebuffer carveouts itself; do not duplicate
  them. The mainline SoC dtsi's static list must be deleted and replaced by an
  empty node.
- Dynamic reserved-memory nodes must fit their `size` into the first
  `alloc-ranges` window inside free DRAM, otherwise LK stops.
- **No `/bootprof` node.** LK creates it after the fixup steps and stops if it already
  exists.
- `kaslr-seed`, if present, must be exactly 8 bytes (section 4).
- A node compatible with `mediatek,sspm` must not carry
  `sspm-res-ram-size = <0>` (zero-sized allocation, LK panics). Leaving the node
  out is fine.
- If an SD card controller uses the same compatible as the eMMC controller
  (`mediatek,mt6878-mmc`), note that when booting from UFS LK sets
  `status = "disabled"` on the **first** node with that compatible; the SD
  controller node must not be the first one.

### 7.3 ADSP shared memory sizes

One fixup step reserves shared memory for the audio DSP. It sums all properties whose
names contain `adsp-rsv` in the node compatible with `mediatek,scp-audio-mbox`
(each rounded up to 4 KiB) and, unless a node compatible with
`mediatek,reserve-memory-adsp_share` exists, allocates the sum (aligned to
64 KiB, 16 MiB alignment, below `0xa0000000`) as `adsp_shared_reserved`. A zero
sum makes the allocation, and LK, fail. The stock values work:

```
scp-audio-mbox {
	compatible = "mediatek,scp-audio-mbox";
	adsp-rsv-logger-a = <0x180000>;
	adsp-rsv-ipidma-a = <0x200000>;
	adsp-rsv-audio = <0x5c0000>;
	adsp-rsv-xhci = <0x80000>;
};
```

LK then reserves 0x9c0000 bytes and writes the address into a `shared_memory`
property of this node. Do not add a node compatible with
`mediatek,snd-audio-dsp`; it only adds another size. The alternative with a
`mediatek,reserve-memory-adsp_share` node was not tested.

### 7.4 Optional nodes

When these are missing LK logs it and skips that step: DRAM controller
info (`dramc`), display debug config, modem driver nodes (`mddriver`,
`md_attr_node`), bus tracker, watchdog, secure memory features, GPU microcontroller
(`gpueb`), audio DSP, battery/gauge, charger, PMIC AUXADC, SSPM, the Type-C
controller, UFS/eMMC host nodes, and the `model` property (LK rewrites `model`
only if it exists).

Useful option, not used yet: a watchdog node with `apwdt_en = <0>` makes LK
switch off the hardware watchdog (TOPRGU at `0x1c00a000`) before starting the
kernel.

## 8. Nodes required by the firmware loaders

In parallel with kernel decompression LK starts a task that loads coprocessor
firmware. This task reads **our** DTB (not LK's built-in copy) and every loader
must return success, otherwise LK stops. This is where boot attempt 1 failed.

In recovery mode almost all loaders are skipped (only the modem and VCP loaders
run); in charger mode the connectivity loader is skipped.

| loader | what it looks for in our DTB | effect when missing / disabled | what it does when enabled |
|---|---|---|---|
| SCP (sensor/system coprocessor) | node compatible with `mediatek,scp`, its `status` and SCP memory layout properties | **missing node or status other than "okay": LK stops** | reserves `SCP-reserved` (35 MiB) and `scp_share`, loads and starts the SCP firmware |
| VCP (video coprocessor) | node compatible with `mediatek,vcp`, `status`, `vcp-support`, sizes | **missing node or status other than "okay": LK stops** | with `vcp-support = <0>` returns immediately (nothing loaded); with non-zero support both `vcp-sram-size` and `vcp-dram-size` must be non-zero |
| APU (AI accelerator) RISC-V | path `/apusys-rv@190e1000`, `/soc/apusys-rv@190e1000`, `/apusys_rv`, `/soc/apusys_rv`; property `status` | **no such node, or a node without `status`: LK stops** | status beginning with "okay" loads APU firmware; any other status logs "not ready" and succeeds |
| CCU (camera) | node compatible with `mediatek,ccu_rproc`, `secured` | logs, succeeds | with `secured = "yes"` loads CCU firmware |
| SPM (power manager) | node compatible with `mediatek,sleep` | logs, succeeds | writes the SPM firmware version; the SPM firmware is handed to ATF in any case |
| modem | `/mddriver` or `/soc/mddriver`, `ref-args` containing `md_disable` | logs, succeeds | the modem firmware is loaded as in stock unless disabled (the disabled path was not tested) |
| GPU microcontroller | `/soc/gpueb` (optional diagnosis mode) | logs | depends on hardware state left by the preloader, not on the DTB |
| connectivity (Wi-Fi/BT/GNSS) | `/consys`, `/gps` and `/wifi` (each at the root or in `/soc`) | if **any** of the three is missing, the loader does nothing and succeeds | **rewrites `compatible` of `/consys`**, loads the connectivity firmware |
| DPM, audio DSP, APU (generic) | not read | succeed | |

Note the asymmetry: for SCP and VCP a status other than "okay" is fatal, while
the APU loader needs a `status` property but accepts "disabled".

Minimal set used in our DTB (no mainline driver binds to any of these
compatibles, so they have no effect on Linux):

```
scp {
	compatible = "mediatek,scp";
	status = "okay";
	/* SCP memory layout properties with the stock values */
};

vcp {
	compatible = "mediatek,vcp";
	status = "okay";
	vcp-support = <0>;	/* VCP stays off */
};

apusys-rv@190e1000 {
	compatible = "mediatek,mt6878-apusys_rv";
	reg = <0x0 0x190e1000 0x0 0x1000>;	/* for dtc only */
	status = "disabled";	/* APU stays off */
};
```

Rules for the SCP node, as enforced by LK: `secure-dump = "enable"` requires a
non-zero `secure-dump-size`; `scp-protect` other than "disable" requires
`secure-dump = "enable"`; the memory table is a list of `<id size align>`
triplets; `memorydump` has exactly five cells. A node with only `compatible` and
`status` also passes, but then the SCP starts with an empty memory table, which
has never been tested. Our tree copies the stock values of these properties.

Do **not** add `/consys` or `/gps`: with all three connectivity nodes present LK
starts loading the connectivity firmware and rewrites `compatible`.

The unavoidable side effect is that the **SCP firmware is always loaded and
running**, as in stock. It owns its peripherals (for example the sensor hub
I2C/I3C controller), so mainline must not touch hardware that stock assigns to
the SCP. The modem firmware is also loaded, as in stock. Should an upstream SCP
node (`mediatek,mt6878-scp`) appear one day, it has to keep `"mediatek,scp"` in
its compatible list and must not be disabled.

## 9. What LK adds to the DTB

- `/chosen`: boot tags (`atag,boot`, `atag,pl-sec`, `atag,lk-sec`,
  `atag,videolfb*` with the framebuffer address, size and panel info,
  `atag,ufs`, `atag,chipid`, `atag,devinfo`, `atag,masp`), `ram_console`,
  `log_store`, crash dump parameters (`mrdump,*`, `aee,*`), `kaslr-seed`,
  `linux,initrd-start/end`, the final `bootargs`.
- Root: `/firmware/android` (compatible, serial number, boot mode, AVB
  parameters), `/bootprof` (boot time stamps), `/debug-kinfo`
  (`google,debug-kinfo`, with a `memory-region`), and the `model*` properties
  if `model` exists. Any node LK might add, such as `/mdproperties`, is created
  only if the corresponding modem node exists.
- `/memory`: `reg` replaced with the full DRAM range, plus a memory map
  property.
- `/reserved-memory`: about 47 nodes named `mblock-<n>-<name>` with `reg`, a
  compatible derived from the name (or `shared-dma-pool`), `no-map` or
  `reusable` as appropriate. Among them (addresses as seen by our kernel):

  | carveout | address | size |
  |---|---|---|
  | pstore (ramoops) | `0x48090000` | 896 KiB |
  | BL31 (ATF) | `0x48800000` | 2 MiB |
  | TEE | `0x70000000` | 60 MiB |
  | SCP | `0xb8000000` | 35 MiB |
  | modem shared memory and modem images | several regions around `0x8a000000`-`0xedffffff` | |
  | framebuffer | `0xfe04f000` | 32448 KiB |
  | GenieZone hypervisor | `0x33dc00000` | 34 MiB |

  This is convenient for mainline: firmware memory is protected automatically.

- **The framebuffer carveout is written without `no-map`.** On arm64,
  `memremap()` with write-combining refuses RAM that is part of the linear map,
  so simple-framebuffer/simpledrm cannot map it. Our kernel carries a small
  patch that marks the reserved node with compatible `mediatek,framebuffer`
  as `no-map` during the early scan (in the kernel log the region then shows up
  as "nomap ... mblock-15-framebuffer").

## 10. Verified boot (vbmeta)

- LK runs AVB slot verification for `boot`, `vendor_boot` and `init_boot` of the
  current slot. On an unlocked device verification errors are allowed, but
  **invalid metadata is fatal** (no slot data is produced and LK stops). This
  follows from libavb semantics and has not been provoked on hardware.
- The stock `vbmeta` describes `boot` through a chained partition (which would
  require an AVB footer in our `boot`) and has hash descriptors with fixed image
  sizes for `vendor_boot` and `init_boot`.
- Therefore `vbmeta_<slot>` is re-flashed **with verification disabled**
  (`fastboot --disable-verity --disable-verification flash vbmeta_<slot> ...`,
  flags = 2). libavb then loads the partitions whole and does not follow
  chains.

## 11. A/B slots and retries

- Slot metadata is the AOSP `bootloader_control` structure at offset 0x800 of
  `misc` (magic `BCAB`; per slot priority, tries remaining, successful flag;
  CRC32).
- `fastboot set_active <slot>` in LK sets the priority and tries of that slot
  and **also switches the UFS boot LUN**, so the matching preloader copy runs.
  On this phone slot B contains older bootloaders (V01.00.13) than slot A
  (V01.00.14). Switching to slot B means running an older preloader and LK; if
  the newer firmware had raised an anti-rollback counter, the older preloader
  might refuse to start, which would leave only BootROM recovery. Check the B
  chain first without flashing anything (`set_active b`,
  `reboot bootloader`, `getvar current-slot`, `getvar version-bootloader`,
  then `set_active a`).
- The preloader decrements the tries counter on every unsuccessful boot and
  eventually rolls back to the other slot. Mainline does not set the successful
  flag, so a mainline slot falls back after a few boots unless Linux writes the
  flag (with a recomputed CRC).
- The boot attempts so far (see boot-attempts.md) were made on **slot A** by
  temporarily replacing `boot_a`, `vendor_boot_a`, `init_boot_a`, `dtbo_a` and
  `vbmeta_a`, and restoring them from a backup afterwards. LK and the preloader
  were never touched.

## 12. Getting out of a boot loop

- Hold **Volume Up + Power** during the reboot loop until LK shows the
  "Select Boot Mode" menu, then select "Fastboot Mode" (Volume Up moves, Volume
  Down selects). From there the original partitions can be flashed back.
- Volume Up alone selects recovery, which boots the same kernel from the same
  slot, so the loop continues.
- A long press of Power reboots the phone; it does not power it off.
- The preloader's USB download mode only echoes bytes on this device (the
  download agent handshake is disabled), and BootROM mode could not be reached
  with keys. Do not flash `preloader`, `lk`, `tee` or `seccfg`, do not relock the
  bootloader, and do not touch `nvram`, `nvdata`, `nvcfg`, `proinfo` or
  `protect1/2` (calibration and radio identity data).
- After restoring Android, read the reason from the LK log in `expdb` (look for
  `ASSERT`, `panic` and `booting linux`) and, if the kernel ran, from
  `/sys/fs/pstore` (the ramoops region at `0x48090000` is shared by both
  kernels).

## 13. Checklist

DTB:

- [ ] raw FDT (or DT table with the wanted blob first), well below 3.5 MiB
- [ ] root and `/reserved-memory` with 2/2 cells, `/reserved-memory` empty except
      for dynamic nodes, with `ranges;`
- [ ] `/chosen` with `bootargs` and an 8-byte `kaslr-seed`
- [ ] `memory@40000000` with `device_type = "memory"`
- [ ] `emicen`, `emiisu`, the `"mediatek, soc-dbg-error-flag"` node, `/wifi`
- [ ] `scp` (okay), `vcp` (okay, `vcp-support = <0>`), `apusys-rv@190e1000`
      (disabled, with a `status`)
- [ ] `scp-audio-mbox` with non-zero `adsp-rsv-*`
- [ ] no `/bootprof`, no `/consys`, no `/gps`, no zero `sspm-res-ram-size`

Images:

- [ ] header v4, addresses from section 2, page size 4096
- [ ] compressed kernel (gzip or legacy LZ4), decompressed size below 124.5 MiB
- [ ] `CONFIG_BOOT_CONFIG=y`, framebuffer `no-map` patch, pstore/ramoops enabled
- [ ] `boot` command line prefixed with 20 spaces; total command line below
      2047 bytes
- [ ] `init_boot` replaced by an image with an empty cpio
- [ ] `dtbo` zeroed
- [ ] `vbmeta` flashed with verification disabled

The titan2-tools repository contains an image builder
(`scripts/make-mainline-images.sh`) that applies these rules and a checker
(`scripts/lk-fdt-check.py`) that validates a DTB or a set of images against
sections 4-8 before anything is flashed.

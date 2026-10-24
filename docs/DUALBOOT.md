# Android and mainline Linux side by side on the Titan 2

A boot menu ([titan2-boot](../../titan2-boot)) runs after the stock bootloader (LK) and starts
either Android or mainline Linux. LK, the preloader and slot B are not touched. Linux gets a
partition of its own, `linux`, carved from the end of `userdata`.

Status (2026-10-24, one phone, crDroid 16 GSI + mainline 7.2): installed and working. The menu,
Android, recovery (TWRP), power-off charging, Linux from the pack and Linux from the `linux`
partition all boot; Linux cannot write to Android's partitions (see "Protection" below).

> **Experimental.** Installing wipes Android's data once. You need an unlocked bootloader, a
> working `fastboot`, TWRP for the Titan 2 in `vendor_boot_a` (titan2-twrp) and backups (below).
> Everything is at your own risk.

## Layout

| partition | content |
|---|---|
| `boot_a` | titan2-boot + a pack: the Android kernel, a fallback mainline kernel/DTB/initramfs, an `extlinux.conf` with the menu entries |
| `vendor_boot_a`, `init_boot_a`, `dtbo_a`, `super` | Android's, unchanged (TWRP lives in `vendor_boot_a`) |
| `userdata` | Android, shorter by 256 GiB (about 210 GiB left on a 512 GB phone) |
| `linux` (new, GPT entry 78) | ext4 with Alpine and `/boot/extlinux/extlinux.conf` |

Why like this:

- LK loads a kernel only from `boot_<slot>`, so the menu lives in `boot_a`.
- **The Android kernel stays in the menu pack.** Recovery, power-off charging, the alarm boot and a
  normal Android boot never depend on the menu's UFS reader: the menu brings the UFS up only when
  "Linux partition" is chosen (or is the default). The price: a new Android kernel means rebuilding
  the pack (`ANDROID_BOOT=<boot.img> scripts/make-bootmenu.sh`) and flashing `boot_a`.
- The Linux kernel on the `linux` partition is updated from Linux itself; the copy in the pack is a
  fallback, e.g. if the menu cannot read the UFS.
- Slot B is never activated (it holds older bootloaders). **Do not install OTA updates**: they write
  slot B and switch to it.

## The menu

Volume up/down moves, power starts the entry (if the PMIC does not answer, volume down starts).
A countdown with a bar runs to the default entry. Entries: Android, `Linux <version> (boot_a)` (the
fallback kernel from the pack, root on the `linux` partition), "Linux partition" (the entries of
`/boot/extlinux/extlinux.conf` on `linux`), Fastboot, Recovery, Reboot. In recovery, charging,
alarm and factory boots the menu is skipped and Android starts at once.

If the menu's UFS reader hangs, it writes its log into Android's pstore first: after the watchdog
reset, Android shows it in `/sys/fs/pstore/console-ramoops-0`.

## Protection of Android's partitions from Linux

Linux writes only to its own partition. The initramfs marks every UFS device (disks and
partitions behind the UFS controller `112b0000.ufshci`, except the root partition) read-only and
then sets the one-way kernel lock `block.ro_lock` (`CONFIG_BLK_DEV_RO_LOCK`, titan2-linux branch
`titan2`). Until the next boot not even root can then make the UFS writable, write to its
partitions or to the whole disk (`dd`, dm, discard, SG_IO, bsg/ufs-bsg), change the partition
table (BLKPG, BLKRRPART) or get a LU back writable by deleting and rescanning it. Root can write
to its root, to an SD card and to USB storage (not affected), and use loop/dm on its own files.
Left open on purpose (needed for debugging): own kernel modules (no module signing), kexec,
`/dev/mem`. A kernel without the lock refuses to mount a root from the partition (unless
`t2.root_unsafe` is given).

## Build (titan2-tools, no phone needed)

```
scripts/build-mainline.sh                                       # kernel, DTBs, modules
AUTHORIZED_KEYS=~/.ssh/id_ed25519.pub scripts/make-linux-root.sh  # out/linux-root/linux.img (sparse, ~450 MB)
git clone https://github.com/Serlk/titan2-boot                  # into the titan2-tools directory
ANDROID_BOOT=<the boot.img of your installed Android> OUT=out/bootmenu-dualboot \
    UFS=1 LINUX_ROOT=part:linux VARIANTS=full DEFAULT=android scripts/make-bootmenu.sh
```

Without SSH keys (`AUTHORIZED_KEYS`, `tools/initramfs/authorized_keys` or
`~/.ssh/id_ed25519.pub`) root has an empty password (SSH over USB only, but the image stays on
the phone). Tests without the phone: `scripts/test-gpt-carve.sh` (the repartitioning on an image
with the phone's real GPT layout), `scripts/test-bootmenu-qemu.sh` (the menu in qemu),
`tools/initramfs/qemu/diskroot.sh` (root on a partition, the lock and write attempts as root).

## Installation

One step at a time; check the result before the next one.

1. **Backups** (from rooted Android or TWRP, `dd if=/dev/block/by-name/<p>`): `boot_a`,
   `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `vbmeta_a`, `misc`, and the partitions with
   calibration data and keys: `protect1`, `protect2`, `nvdata`, `nvcfg`, `nvram`, `proinfo`,
   `seccfg`. Take `nvdata` and friends from TWRP (Android keeps them mounted and writes to them).
   Keep a `SHA256SUMS` next to them.
2. **Menu only:** `fastboot flash boot_a out/bootmenu-dualboot/boot.img`. Check: the menu,
   Android, Recovery, power-off charging, and "Linux partition" (it must show
   `UFS: LU2 476 GiB, GPT ok` and `no partition "linux"`), then Android from the same menu.
3. **Repartition from TWRP** (do not decrypt `/data`; TWRP may attach the metadata decryption
   itself, that is fine as long as nothing is mounted):
   `scripts/dualboot-repartition.sh check`, then `scripts/dualboot-repartition.sh run`. It saves
   the start and end of the disk in `backup/<date>-pre-repartition/`, writes the backup GPT, then
   the primary, reads both back and checks them. The Titan 2 GPT has exactly 77 entries for 77
   partitions; the tool grows the entry count to 78 (the arrays keep their sectors).
4. **Wipe Android's data** in this order (otherwise Android boot-loops from the second boot on):
   `fastboot reboot fastboot`, `fastboot format:f2fs metadata`, `fastboot reboot bootloader`,
   `fastboot -w`.
5. **Linux:** `fastboot flash linux out/linux-root/linux.img`, `fastboot reboot`. Android starts
   from scratch (KernelSU/Magisk managers and apps have to be installed again).
6. **First Linux boot:** "Linux partition" → Linux. The first boot runs the package scripts; then
   `t2-growroot` grows the ext4 to the whole partition in the background. SSH over USB:
   `ssh root@172.16.42.1`.

Notes from the first installation:

- The LK of the Titan 2 answers `fastboot getvar partition-size:` only for a fixed list of
  partitions (`linux` is not in it); `fastboot flash linux` finds the partition.
- TWRP clears the recovery flag in `misc` itself 10–20 s after it starts; rebooting it earlier
  boots straight back into TWRP. See [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md).

## Rollback

See [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md).

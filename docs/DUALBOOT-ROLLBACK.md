# Dual boot: how to undo it

For [DUALBOOT.md](DUALBOOT.md). Commands run in the titan2-tools directory; `<backup>` is the
directory where you saved your partitions before the installation (with `SHA256SUMS`).

## Getting into fastboot (LK)

- From the titan2-boot menu: choose **Fastboot** with the volume keys, start it with power.
- From Android: `adb reboot bootloader`.
- If the phone does not boot, hangs or loops: hold **power for about 15 s** (it turns off), then
  hold **volume UP + power** until the "Select Boot Mode" menu; volume up selects, volume down
  confirms; choose **Fastboot Mode**.
- Check: `fastboot devices` lists the phone as `fastboot`.

## Only the menu was flashed (step 2)

```
PARTS=boot_a scripts/restore-stock-boot.sh <backup>
```
or by hand: `fastboot flash boot_a <backup>/boot_a.img && fastboot reboot`. Nothing else changed,
Android's data is intact. `scripts/restore-stock-boot.sh <backup>` without `PARTS` also restores
`init_boot_a`, `vendor_boot_a` and `dtbo_a` from the same backup (take a backup made while TWRP
was in `vendor_boot_a` if you want to keep TWRP).

## After the repartitioning (steps 3–6)

Android's data is wiped in these steps anyway, so undoing them means putting the old partition
table back and formatting `userdata` again.

1. Boot TWRP: **Recovery** in the menu, or `fastboot reboot recovery` (TWRP lives in
   `vendor_boot_a` and does not depend on the menu).
2. Do not decrypt anything in TWRP (cancel the password prompt), then on the PC:
   ```
   scripts/dualboot-repartition.sh restore backup/<date>-pre-repartition
   ```
   `scripts/dualboot-repartition.sh run` created that directory: `head.bin` and `tail.bin` (64
   sectors from the start and the end of the disk before the change, with `SHA256SUMS`),
   `disk.env` (device, sector size, disk size), `new/carve.env` (`HEAD_SECTORS`, `TAIL_LBA`,
   `TAIL_SECTORS`) and `partitions.txt`. `restore` writes only the GPT sectors: `head.bin` from
   LBA 0 (`HEAD_SECTORS` sectors) and the end of `tail.bin` at `TAIL_LBA`.
3. Wipe in this order: `adb reboot bootloader`, `fastboot reboot fastboot`,
   `fastboot format:f2fs metadata`, `fastboot reboot bootloader`, `fastboot -w`, `fastboot reboot`.
4. If Android does not start after that: TWRP → Wipe → Format Data → yes, reboot.
5. Remove the menu from `boot_a` as above.

If the mobile network, the IMEI or the Wi-Fi MAC is gone, or Android complains about keys (should
not happen: these partitions lie before `userdata` and are not touched), check the backup
(`sha256sum -c SHA256SUMS`) and flash the partition back from fastboot (LK), e.g.
`fastboot flash nvdata <backup>/nvdata.img` (likewise `protect1`, `protect2`, `nvcfg`, `nvram`),
then `fastboot reboot`.

## TWRP boots again and again

That is the recovery flag in `misc`. TWRP clears it itself 10–20 s after it starts (before the
decryption screen), so do not reboot TWRP earlier. If it loops anyway: TWRP → Reboot → System, or
`adb shell dd if=/dev/zero of=/dev/block/by-name/misc bs=2048 count=1` and `adb reboot`.

## Linux booted but something is wrong

Do not repair anything from inside Linux: reboot (`t2-reboot bootloader` over SSH, or a long
press of power) and choose Android. Linux cannot write to Android's partitions (the UFS lock).

## Never

- Never flash or erase the preloader, lk, tee, scp, nvram, nvdata, nvcfg, protect1/2, proinfo,
  seccfg, md1img: they hold calibration data and keys (no network or IMEI without them).
- Never switch the slot (`fastboot set_active b`, `--slot b`): slot B holds older bootloaders.
- Never install Android OTA updates (they write slot B and switch to it).
- A `boot.img` flashed over the menu removes the menu (Android still boots; flash the menu image
  again to get it back).

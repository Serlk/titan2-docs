<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Installing mainline Linux on the Unihertz Titan 2 (dual boot with Android)

[Русская версия](INSTALL-LINUX.ru.md) · Building: [BUILD-LINUX.md](BUILD-LINUX.md) ·
What works: [STATUS.md](STATUS.md) · Details of the dual boot: [DUALBOOT.md](DUALBOOT.md) ·
Undoing it: [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md)

## 1. What you get

- A boot menu after the phone's own bootloader: **Android** or **Linux**, chosen with the
  volume keys. Android, its recovery and power-off charging keep working as before.
- Mainline **Linux 7.2** with an **Alpine Linux** (v3.24, aarch64) system on a partition of its
  own (`linux`, 256 GiB by default, carved from the end of Android's `userdata`).
- On Linux: the screen, keyboard, touch and touchpad, Wi-Fi, mobile data, VoLTE calls and SMS,
  Bluetooth (basic), the GPU (OpenGL ES and Vulkan with our Mesa build), charging, sensors, and
  optionally the cameras and GNSS. The details, and what does not work, are in
  [STATUS.md](STATUS.md).
- Linux **cannot write to Android's partitions**: everything on the UFS except the `linux`
  partition is made read-only with a lock that not even root can lift until the next boot.

This is a project for people who are comfortable with `fastboot`, `adb`, SSH and a shell. There
is no installer and no graphical setup; a desktop environment of our own is being developed and
will be published later. Meanwhile the image ships the **niri** Wayland compositor, which is what
we use on the phone; other Wayland compositors from Alpine (phosh, sxmo, sway, ...) should run on
the same DRM/GPU stack but we have not tested them.

## 2. Risks and disclaimer

- The installation **wipes Android's data once** (the repartitioning needs it). Back up what you
  need from Android first.
- A mistake while flashing can leave the phone unbootable. Everything here was done on our own
  phone and is reversible as long as you keep the backups and never touch the partitions listed
  in section 4.
- Using an unlocked bootloader and custom software may void your warranty and weakens the
  phone's security (anyone with physical access can flash it).
- This is an independent community project, not affiliated with Unihertz or MediaTek. There is
  no warranty of any kind; you do everything at your own risk.

## 3. Requirements

- **Unihertz Titan 2** (MT6878) on stock firmware **V01.00.14**, with the **bootloader
  unlocked** (`fastboot flashing unlock`, which wipes the phone) and verification disabled in
  `vbmeta_a`. The installation of [crDroid for the Titan 2](https://github.com/Serlk/titan2-android)
  takes you through both; this dual boot was tested with crDroid 16 as the Android side. Stock Android should
  work the same way (the menu takes the kernel of whatever Android you have), but we have not
  tried it.
- **Slot A** active: `fastboot getvar current-slot` must say `a`.
- **TWRP for the Titan 2** in `vendor_boot_a`:
  [titan2-twrp](https://github.com/Serlk/titan2-twrp), see its `docs/INSTALL.md`. It is needed for
  the backups and the repartitioning.
- A Linux PC with Docker, git, `adb`/`fastboot` (recent platform-tools), `img2simg`, `ssh`, `e2fsprogs`,
  about 30 GB of free disk (60 GB with the Mesa build) and the images from
  [BUILD-LINUX.md](BUILD-LINUX.md).
- A 512 GB phone: the default `linux` size of 256 GiB leaves Android about 210 GiB. The size
  is a parameter (`SIZE=`, at least 64 GiB must stay for Android).
- A good USB cable, battery above 50 %.

## 4. Never do this

- Never flash or erase `preloader`, `lk`, `tee`, `scp`, `seccfg`, `nvram`, `nvdata`, `nvcfg`,
  `protect1`, `protect2`, `proinfo`, `md1img`: they hold the bootloaders, keys and calibration
  data (without them: no network, no IMEI).
- Never switch to slot B (`fastboot set_active b`, `--slot b`): it holds older bootloaders.
- **Never install Android OTA updates**: they write slot B and switch to it.
- Do not lock the bootloader while anything but signed stock firmware is installed.
- Do not run `fastboot -w` in fastbootd (where `fastboot getvar is-userspace` says `yes`) except
  in the exact order given in step 5.

**Getting into fastboot** at any time: from the boot menu choose **Fastboot**; from Android
`adb reboot bootloader`; from Linux `t2-reboot bootloader`. If nothing boots: hold **Power
for about 15 s** until the phone is off, then hold **Volume Up + Power** until the bootloader
menu ("Select Boot Mode") appears, select **Fastboot Mode** (volume up moves, volume down
confirms). The TWRP documentation describes **Power + Func1** for the same bootloader menu; if
one combination does not bring it up, try the other. `fastboot devices` must list the phone.

## 5. Installation

Do one step at a time and check its result before the next one. Commands run on the PC in the
titan2-tools directory; `<date>` is any name you choose for the backup directory.

### Step 1. Android and TWRP

Install Android (crDroid or stock V01.00.14) and TWRP as their own instructions say. Boot Android
once and make sure it works, then boot TWRP (Recovery in the bootloader menu, or
`adb reboot recovery`). **Do not decrypt `/data` in TWRP** for the following steps (cancel the
password prompt).

### Step 2. Backups

In TWRP, with `adb` working:

```sh
mkdir -p backup/<date>-pre-linux && cd backup/<date>-pre-linux
for p in boot_a init_boot_a vendor_boot_a dtbo_a vbmeta_a misc \
         protect1 protect2 nvdata nvcfg nvram proinfo seccfg; do
    adb pull /dev/block/by-name/$p $p.img
done
sha256sum *.img > SHA256SUMS
cd ../..
```

Keep this directory safe, also outside the PC. `boot_a.img` is the kernel of your Android for
the boot menu; `nvdata`, `nvcfg`, `protect1`, `protect2` are also the source of the modem's
NVRAM copy on Linux (section 9). The backup is personal: it contains your IMEI and keys, never
share it.

### Step 3. Build the images

Follow [BUILD-LINUX.md](BUILD-LINUX.md): the kernel, `out/linux-root/linux.img`, and the boot
menu built with `ANDROID_BOOT=backup/<date>-pre-linux/boot_a.img`
(`out/bootmenu-dualboot/boot.img`).

### Step 4. The boot menu alone

```sh
adb reboot bootloader
fastboot getvar current-slot          # a
fastboot flash boot_a out/bootmenu-dualboot/boot.img
fastboot reboot
```

Check before going on:

1. The menu appears after the bootloader logo and counts down to Android; Android boots.
2. Recovery from the menu starts TWRP; power-off charging still works.
3. "Linux partition" shows `UFS: LU2 ... GiB, GPT ok` and the error `no partition "linux"`
   (the partition does not exist yet); then Android from the same menu still boots.

If anything here fails: `fastboot flash boot_a backup/<date>-pre-linux/boot_a.img` brings back
the state before this step (nothing else has changed yet).

### Step 5. Repartition and wipe

Boot TWRP, do not decrypt, then on the PC:

```sh
scripts/dualboot-repartition.sh check     # reads and checks the GPT, writes nothing
scripts/dualboot-repartition.sh run       # saves the GPT, carves "linux", writes after "yes"
```

`run` saves the start and the end of the disk in `backup/<date>-pre-repartition/` (needed for
the rollback), writes the backup GPT, then the primary one, and reads both back. `SIZE=128GiB`
before the command picks another size (default 256GiB). Only the partition table changes;
`userdata` gets shorter and Android's data is wiped next.

Wipe Android's data **in this order** (another order makes Android boot-loop from its second
boot on):

```sh
adb reboot bootloader
fastboot reboot fastboot
fastboot format:f2fs metadata
fastboot reboot bootloader
fastboot -w
```

### Step 6. Flash Linux

```sh
fastboot flash linux out/linux-root/linux.img
fastboot reboot
```

The bootloader does not report the size of the new partition (`getvar partition-size:linux`
fails), but `fastboot flash linux` finds it. Android now starts from scratch: set it up again
(root managers and apps have to be reinstalled).

## 6. First boot of Linux

1. Reboot, choose **Linux partition** in the menu, then the entry **Linux ... -all** (the
   default of a fresh image is the base entry `Linux <version>`, i.e. `full`, which has no
   Wi-Fi; `all` has everything).
2. The first boot runs the package scripts and grows the file system to the whole partition in
   the background; it takes a few minutes. Text console output appears on the screen.
3. Connect the USB cable. The phone appears as a USB network adapter and gives the PC an address
   (172.16.42.2) by DHCP:
   ```sh
   ssh root@172.16.42.1
   ```
4. On the phone:
   ```sh
   passwd                                   # a root password for the console
   nmcli device wifi list
   nmcli device wifi connect "<SSID>" password "<password>"
   date                                     # chrony sets the time once there is a network
   ```
   Wi-Fi comes back by itself on later boots; the phone is then also reachable over SSH on its
   Wi-Fi address.

Optional, internet for the phone through the PC over USB: the phone routes through 172.16.42.2
with a lower priority than Wi-Fi. On the PC this needs forwarding and NAT, for example
`sysctl -w net.ipv4.ip_forward=1` and
`iptables -t nat -A POSTROUTING -s 172.16.42.0/24 -j MASQUERADE` (undo them when you are done).

## 7. The boot menu

- **Volume up/down** move, **Power** starts the entry (if the power key is not read, Volume
  Down starts it). A countdown runs to the default entry.
- Entries: **Android**; **Linux <version> (boot_a)**: a fallback kernel stored in `boot_a` with
  the root on `linux`; **Linux partition**: the entries of `/boot/extlinux/extlinux.conf` on
  the `linux` partition (`all`, `all-fb`, `full`, test variants, `previous`); **Fastboot**;
  **Recovery**; **Reboot**.
- Recovery, power-off charging and alarm boots skip the menu and start Android directly.
- The default of the pack is set when it is built (`DEFAULT=android|linux|ufs`, `TIMEOUT=`;
  `DEFAULT=ufs` boots straight into the partition's default entry). The default inside "Linux
  partition" is the `DEFAULT <label>` line of `/boot/extlinux/extlinux.conf`; edit it on the
  phone, or let `linux-update.sh` set it (`DEFAULT=all`).
- Only the first entries fit on the screen (12 or more, depending on the titan2-boot version):
  `linux-update.sh` puts `all`, `all-fb`, `full` first; `VARIANTS="..."` limits the list.
- If the menu's UFS reader hangs, hold Power for 15 s; Android keeps working (it does not use
  the reader), and its log is in Android's `/sys/fs/pstore/console-ramoops-0` after the reset.
- A new Android kernel (an Android update that changes `boot`) means: rebuild the menu with
  `ANDROID_BOOT=<the new boot.img>` and flash `boot_a`. Flashing an Android `boot.img` over
  `boot_a` removes the menu (Android still boots; flash the menu again to get it back).

## 8. Updating the kernel

Build the new kernel on the PC (`scripts/build-mainline.sh`), boot the phone into Linux from
the partition, connect it over USB (or Wi-Fi), then:

```sh
scripts/linux-update.sh                       # HOST=root@172.16.42.1 by default
HOST=root@<wifi-address> scripts/linux-update.sh
REBOOT=1 scripts/linux-update.sh              # and reboot afterwards
```

It copies the kernel, every DTB, a fresh initramfs and the modules to the phone and rewrites
`/boot/extlinux/extlinux.conf` (entries `all`, `all-fb`, `full`, the other variants, and
`previous`). The kernel, DTBs and initramfs that were there before move to `/boot/prev`
(entry **previous**), and old module trees are kept, so a kernel that does not boot is one menu
entry away from the old one. Entries of other kernels you put under `/boot/<name>/` are kept.
Nothing is flashed; `boot_a` and Android are not touched.

Options: `KO=<build dir>`, `VARIANTS="all all-fb full ..."`, `DEFAULT=all`, `CMDLINE=...`,
`SSH_OPTS="-i <key>"`, `FIRMWARE=1` (section 9). Kernel parameters that must survive updates go
into `/boot/extlinux/cmdline.extra` on the phone (one line); the script appends it to every
entry. Run the script unfiltered: piping it into `head` kills it halfway through.

The fallback kernel inside `boot_a` is only updated by rebuilding and reflashing the menu; it
does not need to follow every update (it keeps working with its own modules on the partition).

## 9. Factory files from your own phone

Some parts need files that belong to your phone: firmware and tuning of the vendor, and
per-device calibration. **They are never distributed by this project, and you must not share
yours.** You copy them from your own phone (or from the official firmware package of your own
firmware version), for use on that same phone.

| what | needed for | source |
|---|---|---|
| `frsm-spk1.bin` | the loudspeaker amplifier | `vendor/firmware/` of the vendor partition |
| `WIFI` (Wi-Fi MAC address and RF calibration) | Wi-Fi with your own MAC and calibration | `/APCFG/APRDEB/WIFI` in `nvdata` |
| `BT_Addr` | your Bluetooth address | `/APCFG/APRDEB/BT_Addr` in `nvdata` |
| modem NVRAM copy | the modem (IMEI, calibration) | the `nvdata`, `nvcfg`, `protect1`, `protect2` backups of step 2 |
| vendor, system, system_ext files | GNSS, fingerprint, camera | the `vendor_a`, `system_a`, `system_ext_a` logical partitions of `super` |

The Wi-Fi/BT radio firmware, the modem firmware and the sensor hub firmware are loaded by the
phone's bootloader from its own partitions, exactly as for Android; Linux does not need copies.

**The vendor partitions.** In TWRP (or a rooted Android). `adb shell ls /dev/block/mapper` must
list `vendor_a`; if it does not, mount Vendor in TWRP's Mount screen once (TWRP then maps it).

```sh
mkdir -p out/stock-super && cd out/stock-super
adb pull /dev/block/mapper/vendor_a vendor_a.img      # also system_a, system_ext_a if needed
# rooted Android instead of TWRP:
#   adb exec-out su -c 'cat /dev/block/mapper/vendor_a' > vendor_a.img
fsck.erofs --extract=vendor vendor_a.img            # erofs-utils; for an ext4 image use debugfs rdump
cd ../..
```

Alternatively take the whole `super` partition (`adb pull /dev/block/by-name/super`, several GB,
or the `super.img` of the official package of your firmware version) and split it with
`lpunpack super.img`. Note: if you run crDroid, `system_a` is crDroid's; the parts that need
stock system files (fingerprint, GNSS) need them from the stock package or a dump made before.

**Audio, Wi-Fi and Bluetooth files** onto the phone: `linux-update.sh` can do it
(`FIRMWARE=1` takes `frsm-spk1.bin` from `out/*-super/*/firmware/`, i.e. from the extraction
above, and `WIFI`/`BT_Addr` from the newest `backup/*/nvdata.img`, or `NVDATA_IMG=<file>`), or
by hand:

```sh
debugfs -R "dump /APCFG/APRDEB/WIFI WIFI" backup/<date>-pre-linux/nvdata.img
debugfs -R "dump /APCFG/APRDEB/BT_Addr BT_Addr" backup/<date>-pre-linux/nvdata.img
ssh root@172.16.42.1 mkdir -p /lib/firmware/mediatek/mt6878
scp WIFI BT_Addr root@172.16.42.1:/lib/firmware/mediatek/mt6878/
scp out/stock-super/vendor/firmware/frsm-spk1.bin root@172.16.42.1:/lib/firmware/
```

## 10. Mobile network, calls and SMS

The modem runs with its own firmware (loaded by the bootloader); Linux has to answer its file
requests from a copy of the phone's NVRAM, start it, and hand it to ModemManager. This is not
packaged yet and takes some manual steps. The full reference for the ModemManager part is
`tools/mm-titan2/INSTALL.md` in titan2-tools.

1. **Kernel parameters.** On the phone:
   ```sh
   echo "mtk_ccci.power_ctrl=1 mtk_ccci.runtime_data=1 mtk_ccci.wwan_ports=1" > /boot/extlinux/cmdline.extra
   ```
   then run `scripts/linux-update.sh` once from the PC (it adds the line to every entry) and
   reboot into `all`.
2. **The NVRAM sandbox** (a copy; the modem never writes to a flash partition). On the PC, from
   the backup of step 2 (needs root for the loop mounts):
   ```sh
   B=backup/<date>-pre-linux; S=out/modem-sandbox/ro; mkdir -p $S /tmp/t2m
   for p in nvdata:nvdata protect1:protect_f protect2:protect_s nvcfg:nvcfg; do
       sudo mount -o ro,loop,noload $B/${p%%:*}.img /tmp/t2m
       mkdir -p $S/${p#*:} && sudo cp -a /tmp/t2m/. $S/${p#*:}/
       sudo umount /tmp/t2m
   done
   sudo tar -C out/modem-sandbox -czf modem-sandbox.tgz ro && sudo rm -rf out/modem-sandbox
   scp modem-sandbox.tgz root@172.16.42.1:/root/
   ```
   On the phone:
   ```sh
   adduser -S -D -H -h /var/lib/titan2-modem t2fsd
   mkdir -p /var/lib/titan2-modem && tar -C /var/lib/titan2-modem -xzf /root/modem-sandbox.tgz
   chown -R t2fsd /var/lib/titan2-modem && rm /root/modem-sandbox.tgz
   ```
   The sandbox is your IMEI and calibration: keep it on the phone, do not share it.
3. **The modem servers.** On the phone, with a checkout of titan2-tools (git, or `scp -r`):
   ```sh
   apk add build-base alsa-lib-dev git
   make -C titan2-tools/tools/ccci-fsd && make -C titan2-tools/tools/ccci-rpcd
   make -C titan2-tools/tools/speechd
   install -m 755 titan2-tools/tools/ccci-fsd/t2-ccci-fsd titan2-tools/tools/ccci-rpcd/t2-ccci-rpcd \
       titan2-tools/tools/speechd/t2-speechd /usr/local/bin/
   ```
   In `/etc/conf.d/t2-modem` set `T2_MODEM_RADIO=` (empty: ModemManager switches the radio on),
   then:
   ```sh
   rc-service t2-modem start                  # waits for "modem state: ready"
   head -1 /sys/kernel/debug/d180000.mddriver/fsm
   rc-update add t2-modem default
   ```
4. **ModemManager with the mtk-ccci plugin**: the packages, the plugin build
   (`tools/mm-titan2/build-on-phone.sh` on the phone, or the cross build), the udev rule and the
   sysctl file as described in `tools/mm-titan2/INSTALL.md`. Then:
   ```sh
   rc-update add modemmanager default; rc-service modemmanager start
   rc-update add t2-speechd default;   rc-service t2-speechd start
   mmcli -m any                               # SIMs, registration, access technology
   ```
5. **Mobile data** through NetworkManager:
   ```sh
   nmcli connection add type gsm ifname '*' con-name mobile gsm.apn <your APN>
   nmcli connection up mobile
   ```
6. **Calls and SMS** with `mmcli` (or any ModemManager front end):
   ```sh
   mmcli -m any --voice-create-call="number=<number>"   # prints the call path, e.g. .../Call/1
   mmcli -o 1 --start                                         # dial;  --hangup to end
   mmcli -m any --voice-list-calls; mmcli -o <n> --accept    # incoming
   mmcli -m any --messaging-create-sms="text='Hello',number='<number>'"
   mmcli -s <n> --send; mmcli -m any --messaging-list-sms
   ```
   Call audio is switched by WirePlumber to the card's "Voice Call" profile, so PipeWire and
   WirePlumber must run in your session (the niri session of section 11 starts them).

The network preference defaults to 5G/LTE first (`mmcli -m any --set-allowed-modes=... 
--set-preferred-mode=...` changes it). VoLTE needs LTE; IMS is set up by the plugin.

## 11. Desktop

```sh
adduser <user>                                       # on the phone
for g in video input seat audio; do adduser <user> $g; done
# /etc/conf.d/t2-niri: T2_NIRI_USER=<user>
rc-update add t2-niri default; rc-service t2-niri start
```

`t2-niri` starts niri with a D-Bus session, PipeWire and WirePlumber on the phone's screen.
niri's configuration is in `~/.config/niri/config.kdl`. Keyboard notes: the Titan 2 keyboard has
no Esc key; **Sym** reaches Wayland as `KEY_PROG3` (XKB `XF86Launch3`) and **Fn** as
`KEY_PROG4` (`XF86Launch4`), so bind them in your compositor. `T2_GSK_RENDERER=gl` in
`/etc/conf.d/t2-niri` makes GTK 4 programs use OpenGL ES instead of Vulkan if they misbehave.

## 12. Optional: camera, GNSS, fingerprint

Each of these lives in its own repository with its kernel patch series, its user space and its
own README (how to apply the series, the config fragment, the packages, the services). They are
kept apart on purpose. All of them depend on files from **your own phone** (section 9), which
stay on your phone: firmware and tuning of the vendor are not part of any of our repositories
or images.

| part | repository | state | what you need |
|---|---|---|---|
| Cameras (rear IMX766 and Hi-847, front S5KGD1SP) | [titan2-camera](https://github.com/Serlk/titan2-camera) | works: photos with GNOME Snapshot / qcam / `cam`, software image processing; no video recording yet | kernel with the camera series; libcamera from `scripts/build-libcamera.sh`; root image with `CAMERA=1`; for the front sensor an initialisation table made from your own firmware (the repository README says how) |
| GNSS + A-GPS | [titan2-gnss](https://github.com/Serlk/titan2-gnss) | works: fix, position through GeoClue; A-GPS data downloaded at run time; time to first fix not measured yet | kernel with the GNSS series; the vendor runtime from titan2-tools; the GNSS daemon and HAL from your own `vendor_a`/`system_a` |
| Fingerprint | [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | **not usable yet**: the TEE and the HAL start, but the HAL does not react to the touch interrupt, so no finger is detected | kernel with the fingerprint series; the vendor runtime; the TEE client and HAL from your own stock `vendor_a`/`system_a` |

The vendor runtime (`t2-vendor-setup`, from titan2-tools) builds a private directory
(`/var/lib/titan2-vendor`) out of the extracted partitions of section 9, for example
`t2-vendor-setup /root/stock` where `/root/stock` holds `system/`, `system_ext/` and `vendor/`.
Nothing is downloaded, and the result must not be copied to anyone else's phone.

## 13. Going back

- **Only the menu** (Android untouched): `fastboot flash boot_a backup/<date>-pre-linux/boot_a.img`.
- **The whole dual boot** (old partition table, Android wiped again): from TWRP,
  `scripts/dualboot-repartition.sh restore backup/<date>-pre-repartition`, then the wipe order of
  step 5, then the menu as above. Full procedure: [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md).
- **Fully back to stock**: after the above, flash the official V01.00.14 package with the vendor's
  flash tool (download mode, **not** "Format All"), or flash its `super`, `vbmeta*`, `boot`,
  `init_boot`, `vendor_boot`, `dtbo` images with fastboot and run the wipe. Lock the bootloader
  only after that, if at all.
- If the network, IMEI or Wi-Fi MAC are gone (should not happen: those partitions are never
  written): check the backup with `sha256sum -c SHA256SUMS` and flash the partition back from
  fastboot, e.g. `fastboot flash nvdata backup/<date>-pre-linux/nvdata.img`.

## 14. Troubleshooting

**The screen stays black after choosing Linux.** Wait two or three minutes on the first boot.
Try SSH over USB: if it works, the system is up and only the display is the problem; reboot and
choose `all-fb` (bootloader framebuffer) or `full`. If a new kernel does not boot, choose
`previous`. Reboot a hung phone by holding Power for 15 s.

**No menu, Android starts directly.** Recovery, charging and alarm boots skip the menu by design.
Otherwise `boot_a` no longer has the menu (an Android flash or update wrote it): flash
`out/bootmenu-dualboot/boot.img` again.

**"Linux partition" reports an error.** `no partition "linux"`: the repartitioning is not done
(or was undone). A UFS error: use `Linux ... (boot_a)`, it mounts the same root with the kernel's
own driver. Hangs on "Starting the UFS...": hold Power, Android is not affected.

**No USB network.** Use a data cable; on the PC `dmesg` must show a `cdc_ncm` device. The
initramfs waits up to 60 s for the root partition and stays reachable over USB (SSH) if it does
not find it. Some PCs need the new wired connection set to DHCP ("automatic").

**No Wi-Fi.** The `full` entry has no Wi-Fi: boot `all`. Then `nmcli radio wifi on`,
`rfkill list`, `dmesg | grep -iE 'mt7921|conninfra'`. Without the `WIFI` file (section 9) the
calibration and MAC address of your phone are missing.

**Modem not registering.** `head -1 /sys/kernel/debug/d180000.mddriver/fsm` must say
`state: ready`. If not: the three `mtk_ccci.*` parameters missing in `/proc/cmdline`, no sandbox
in `/var/lib/titan2-modem/ro`, no user `t2fsd`, or the servers not in `/usr/local/bin`
(`/var/log/t2-modem/` has their logs). After a failed start the modem state machine stays
`failed` until a reboot. With the modem ready: `mmcli -m any` and the ModemManager log
(`ModemManager --debug`); `mmcli -m any -e` enables it. VoLTE calls need LTE coverage and IMS
registration; mobile data needs the right APN.

**No sound from the loudspeaker.** `frsm-spk1.bin` missing in `/lib/firmware` (section 9).

**Bluetooth does not appear.** Its driver is not loaded automatically yet:
`modprobe btmtkbgf`, then `bluetoothctl`.

**TWRP starts again and again.** TWRP clears the recovery flag 10-20 s after it starts; do not
reboot it earlier. Otherwise TWRP, Reboot, System.

**The battery drains in sleep.** Known: sleep works but draws about 80 mA, see
[STATUS.md](STATUS.md).

**Something is wrong inside Linux.** Do not try to repair Android from Linux (it cannot write
there anyway). Reboot (`t2-reboot bootloader` gives you fastboot) and use Android or TWRP.

Logs you post publicly (`dmesg`, `mmcli`) contain your serial number, IMEI, SIM identifiers
and MAC addresses; remove them first.

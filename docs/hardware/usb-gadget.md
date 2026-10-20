<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# USB: device mode and the initramfs console

## Hardware

| item | value |
|---|---|
| controller | MediaTek SSUSB (mtu3 device + xHCI host), MAC `0x11201000`, IPPC `0x11203e00` |
| PHY | T-PHY v2 at `0x11f40000`, port u2port (USB 2.0) |
| speed | high-speed (USB 2.0), as in stock |
| PHY supply | MT6369 vusb (3.3 V class), always on |
| PHY tuning (from stock) | `mediatek,eye-vrt = <2>`, `mediatek,discth = <9>` |
| role detection | Type-C controller in the MT6375 (not supported yet) |

The fields used for eye-diagram and disconnect-threshold tuning are at the same
positions in the stock PHY and the mainline T-PHY v2 driver. The efuse-based
PHY calibration is not carried over.

## Power state at kernel entry

The preloader enumerates USB on every boot (looking for a flashing tool), so
the SSUSB power domain, its clocks and VUSB are on before LK. LK's fastboot code
accesses the IPPC and MAC registers without touching the power controller,
i.e. it relies on the domain being on as well. Our DT therefore does not
reference the SSUSB power domain while the power controller is disabled; a
variant DT with power domains does.

## Kernel configuration

- `dr_mode = "peripheral"`, `maximum-speed = "high-speed"`.
- The PHY forces the device role and VBUS-valid (`mediatek,software-role-switch`);
  stock does the same through a forced-VBUS setting in the MAC.
- Fix in the T-PHY driver: when no USB role switch was found through the
  graph port, the delayed work for software role switching was never
  initialised but still queued at power-on, which crashed about 100 ms after
  the controller probed. The work is now always initialised, and without a
  controller the PHY uses a fixed device role.
- A harmless warning remains: fw_devlink cannot create the reverse link
  between the PHY graph port and the controller (dependency cycle).

## The initramfs

A small initramfs (about 1.7 MiB compressed) built from official Alpine Linux
aarch64 packages (busybox, dropbear, udhcpd), verified against the signed
package index.

- USB gadget through configfs: vendor/product 1d6b:0104, **NCM** network
  function plus **ACM** serial. The phone is `172.16.42.1/24`; a DHCP server
  hands the PC exactly `172.16.42.2`, with no gateway or DNS.
- SSH (dropbear) on port 22. Host keys are generated in RAM on each boot. If
  the image contains `/root/.ssh/authorized_keys`, only key login is allowed;
  otherwise root has an **empty password**, which is acceptable only for this
  cable-attached debug image.
- A root shell on the ACM serial port after pressing Enter.
- **Storage is read-only**: no block device is mounted by the scripts; every
  block device is set read-only (BLKROSET) on hotplug and by a periodic sweep;
  `mount` is wrapped to allow only read-only mounts of block devices and to
  refuse calibration, identity, security and user-data partitions entirely.
- Safety net: if no SSH session and no serial login happened within 10 minutes,
  the phone reboots (back to the bootloader and, eventually, the slot
  fallback). Disabled with `t2.noreboot`; the delay is set with
  `t2.reboot_after=`.
- Progress markers go to the kernel log and therefore to ramoops, so they can
  be read from `/sys/fs/pstore` after returning to Android.

## On the PC

```
lsusb | grep 1d6b:0104
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@172.16.42.1
picocom /dev/ttyACM0        # serial console, press Enter
```

Without NetworkManager/DHCP on the PC: assign `172.16.42.2/24` to the new
`cdc_ncm` interface manually. Use `scp -O` (dropbear has no SFTP server).

Connect to a PC directly, not through an OTG hub: the role is forced to device
and Type-C is not handled yet.

<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# titan2-docs

[Русская версия](README.ru.md)

Documentation for **mainline Linux 7.2 on the Unihertz Titan 2** (MediaTek MT6878 /
Dimensity 7300): how to build it, how to install it next to Android, what works, and how the
hardware and the stock bootloader behave. The phone's stock bootloader (LK) is used as it is,
never modified.

## Start here

- [docs/STATUS.md](docs/STATUS.md) - what works, what partly works and what does not (calls,
  SMS, mobile data, Wi-Fi, GPU, charging, sleep, cameras, GNSS, fingerprint, ...)
- [docs/BUILD-LINUX.md](docs/BUILD-LINUX.md) - building everything from source: kernel, DTBs,
  initramfs, the Alpine root file system, the boot menu, optional Mesa and libcamera
- [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md) - installing it as a second system next to
  Android: backups, the boot menu, the `linux` partition, first boot, SSH and Wi-Fi, kernel
  updates, files from your own phone, modem and calls, optional parts, going back,
  troubleshooting

In short (2026-11-12): VoLTE calls, SMS, LTE data, Wi-Fi, the display with the GPU, charging,
sensors, cameras and GNSS work; sleep works but draws about 80 mA; the fingerprint does not work
yet. A desktop environment of our own is being developed and will be published later.

## Documents

Installation and status:

- [docs/STATUS.md](docs/STATUS.md), [docs/BUILD-LINUX.md](docs/BUILD-LINUX.md),
  [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md) (also in Russian: `*.ru.md`)

Boot:

- [docs/lk-boot-requirements.md](docs/lk-boot-requirements.md) - what the stock
  LK requires from `boot`, `vendor_boot`, `init_boot`, `dtbo`, `vbmeta` and the
  DTB (header version, load addresses, command line quirk, ramdisk fragments,
  required and forbidden nodes, firmware loader nodes, slots, recovery)
- [docs/boot-attempts.md](docs/boot-attempts.md) - boot attempts 1-4 of the first alpha: what
  was tried, what happened, what fixed it
- [docs/DUALBOOT.md](docs/DUALBOOT.md) - Android and mainline Linux side by
  side: the titan2-boot menu, the `linux` partition, protection of Android's
  partitions, installation; [docs/DUALBOOT-ROLLBACK.md](docs/DUALBOOT-ROLLBACK.md)
  - how to undo it

Hardware:

- [docs/hardware/overview.md](docs/hardware/overview.md) - SoC, memory, buses
  and device map
- [docs/hardware/display.md](docs/hardware/display.md) - command-mode DSI panel,
  pipeline registers, frame triggering
- [docs/hardware/keyboard-aw9523b.md](docs/hardware/keyboard-aw9523b.md) -
  keyboard matrix, scanning, keymap
- [docs/hardware/touch-rmi4.md](docs/hardware/touch-rmi4.md) - touchscreen and
  keyboard touchpad (RMI4 parameters)
- [docs/hardware/buttons.md](docs/hardware/buttons.md) - power, volume, Func1,
  Func2
- [docs/hardware/spmi.md](docs/hardware/spmi.md) - SPMI controller, RCS
  interrupts, EINT table
- [docs/hardware/pmic.md](docs/hardware/pmic.md) - MT6363, MT6369, MT6319,
  MT6685
- [docs/hardware/charger-mt6375.md](docs/hardware/charger-mt6375.md) - MT6375
  SubPMIC charger
- [docs/hardware/fuel-gauge-sh366100.md](docs/hardware/fuel-gauge-sh366100.md) -
  SH366100 fuel gauge
- [docs/hardware/power-domains.md](docs/hardware/power-domains.md) - SCPSYS
  power domains
- [docs/hardware/clocks.md](docs/hardware/clocks.md) - clock controllers
- [docs/hardware/usb-gadget.md](docs/hardware/usb-gadget.md) - USB device mode
  and the debug initramfs

## Repositories

| repository | what it has |
|---|---|
| [titan2-linux](https://github.com/Serlk/titan2-linux) | the kernel (mainline 7.2 based) and the device trees; branch `titan2` |
| [titan2-tools](https://github.com/Serlk/titan2-tools) | build scripts, initramfs, the Alpine root file system builder, services and user space tools (modem servers, call audio, ModemManager plugin, vendor runtime) |
| [titan2-boot](https://github.com/Serlk/titan2-boot) | the boot menu (Android / Linux) that runs after the stock LK |
| [titan2-docs](https://github.com/Serlk/titan2-docs) | this documentation |
| [titan2-twrp](https://github.com/Serlk/titan2-twrp) | TWRP recovery for the Titan 2 |
| [titan2-camera](https://github.com/Serlk/titan2-camera) | optional: camera drivers and libcamera support |
| [titan2-gnss](https://github.com/Serlk/titan2-gnss) | optional: GNSS and A-GPS |
| [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | optional: fingerprint (not working yet) |
| [titan2-android](https://github.com/Serlk/titan2-android) | crDroid 16 (Android) for the Titan 2 |

## Warning

Flashing anything to a phone can make it unbootable. Never flash `preloader`,
`lk`, `tee` or `seccfg`, and never touch calibration partitions (`nvram`,
`nvdata`, `nvcfg`, `proinfo`, `protect1/2`). Keep a full backup and read how to
get back to Android ([docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md), sections 4 and 13,
and [docs/DUALBOOT-ROLLBACK.md](docs/DUALBOOT-ROLLBACK.md)) before trying anything.

## Disclaimer

This is an independent community project. It is not affiliated with,
endorsed by or supported by Unihertz, MediaTek or any other vendor. All
product names and trademarks belong to their owners. The documents describe
observed behaviour and hardware facts in our own words; they contain no vendor
code or binaries. Use at your own risk.

## How this project is made / affiliation

- This project is developed with the help of AI coding assistants (Anthropic Claude). Commits
  carry a `Co-Authored-By: Claude` trailer; every change is reviewed and tested on the device
  by the human maintainer (serlk), who is responsible for it.
- It is **not affiliated with postmarketOS** or MT6878-Mainline. We build on their GPL kernel
  work (thanks!), but postmarketOS does not accept AI-assisted contributions, so nothing from
  here is submitted to postmarketOS, and this is not a postmarketOS port.

## Sources

The facts in these documents come from the phone itself: its boot logs, register
and bus readings on the device, the factory device tree, and the phone's own
kernel modules, which are declared GPL, and an analysis of the behaviour of the
stock boot loader (no boot loader code is reproduced). Public MediaTek GPL/BSD
sources and upstream Linux were used for comparison. No proprietary code,
firmware or blobs are included.

## License

GPL-2.0-or-later, see [LICENSE](LICENSE). Copyright (c) 2026 serlk.

Licensing: [LICENSE](LICENSE) (GPL-2.0), per-file details in [LICENSING.md](LICENSING.md).

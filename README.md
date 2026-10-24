<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# titan2-docs

[Русская версия](README.ru.md)

Documentation for the **alpha release of mainline Linux 7.2 on the Unihertz
Titan 2** (MediaTek MT6878 / Dimensity 7300), booted by the phone's stock LK
bootloader without modifying it.

This repository contains documentation only: the boot interface of the stock
bootloader, the results of the first boot attempts, and hardware descriptions
gathered while bringing up the mainline kernel.

## State of the alpha (boot attempt 4)

Works: screen (simpledrm on the LK framebuffer plus a frame trigger for the
command-mode DSI panel), QWERTY keyboard (AW9523B matrix), touchscreen and
keyboard touchpad (Synaptics RMI4), volume keys, Func2, UFS (read-only),
USB device mode with NCM + ACM and SSH, MT6363/MT6369 regulators.

Does not work yet: power key and Func1 (PMIC interrupt, fix pending), reading
`clk_summary` hangs the bus; charger, fuel gauge, audio, modem, Wi-Fi,
camera and GPU are not enabled.

## Documents

Boot:

- [docs/lk-boot-requirements.md](docs/lk-boot-requirements.md) - what the stock
  LK requires from `boot`, `vendor_boot`, `init_boot`, `dtbo`, `vbmeta` and the
  DTB (header version, load addresses, command line quirk, ramdisk fragments,
  required and forbidden nodes, firmware loader nodes, slots, recovery)
- [docs/boot-attempts.md](docs/boot-attempts.md) - boot attempts 1-4: what was
  tried, what happened, what fixed it
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

## Related repositories

- **titan2-linux** - the kernel patches and the device tree (mainline 7.2 based)
- **titan2-tools** - image builder (`scripts/make-mainline-images.sh`), DTB
  checker for the LK requirements (`scripts/lk-fdt-check.py`), initramfs and
  helper scripts

## Warning

Flashing anything to a phone can make it unbootable. Never flash `preloader`,
`lk`, `tee` or `seccfg`, and never touch calibration partitions (`nvram`,
`nvdata`, `nvcfg`, `proinfo`, `protect1/2`). Keep a full backup and read
[docs/lk-boot-requirements.md](docs/lk-boot-requirements.md) (sections 11-12) on
how to get back to Android before trying anything.

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

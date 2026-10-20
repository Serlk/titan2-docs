<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Boot attempts 1-4

All attempts took place on 2026-10-20, with mainline Linux 7.2 (MT6878-Mainline
base plus the Titan 2 branch), built with clang 21. Each attempt temporarily
replaced `boot_a`, `vendor_boot_a`, `init_boot_a`, `dtbo_a` and `vbmeta_a`
(verification disabled) on slot A; the stock LK and preloader were not
modified. After each attempt the original partitions were restored from a
backup through LK fastboot (Volume Up + Power during the reboot loop, then
"Fastboot Mode").

The initramfs of every attempt brings up a USB gadget (NCM network + ACM
serial) with an SSH server and makes every block device read-only, so that
nothing on UFS can be written (see hardware/usb-gadget.md).

Requirement numbers below refer to docs/lk-boot-requirements.md.

## Summary

| # | image variant | result | cause | fix |
|---|---|---|---|---|
| 1 | minimal DTB (no SPMI), USB initramfs | boot loop, kernel never started | LK firmware loader for the SCP found no `mediatek,scp` node with status "okay" and stopped | add `scp`, `vcp` and `apusys-rv` nodes for LK (section 8) |
| 2 | as 1 plus the loader nodes | boot loop, kernel never started | LK fixup step for the audio DSP tried to reserve 0 bytes (`adsp_shared_reserved`) and panicked | add `scp-audio-mbox` with the stock `adsp-rsv-*` sizes (section 7.3) |
| 3 | minimal DTB, USB initramfs | **kernel boots**, SSH over USB works, screen does not update | panel is a DSI command-mode panel; writes to the framebuffer never reach it | frame trigger driver `mtk-cmdfb-kick` (hardware/display.md) |
| 4 | full DTB (SPMI, PMICs, input), USB initramfs | **boots, screen, keyboard, touch, USB, SSH** | power key and Func1 do not work (PMIC interrupt not delivered) | pinctrl EINT table fix, pending test |

## Attempt 1: stopped in LK's firmware loader

- Symptom: endless reboot, nothing on screen after the LK logo, pstore empty.
- LK log (`expdb`): an assertion that the SCP loader returned an error.
- Cause: LK loads coprocessor firmware based on **our** DTB, between kernel
  decompression and the DTB fixup steps. Earlier work had only covered the
  fixup steps. The SCP and VCP loaders treat a missing node, or a status other than
  "okay", as fatal; the APU loader needs a node with a `status` property.
- Also seen in the same log: two boots through recovery mode (Volume Up held)
  stopped because our `vendor_boot` has no RECOVERY ramdisk fragment
  (section 5). Normal boot is unaffected.
- Fix: the LK compatibility nodes `scp` (okay, stock memory layout), `vcp`
  (okay, `vcp-support = <0>`) and `apusys-rv@190e1000` (disabled). A DTB checker
  was extended to flag missing loader nodes before flashing.

## Attempt 2: stopped in an LK fixup step

- Symptom: same boot loop.
- LK log: the firmware loaders passed; a DTB fixup step panicked in LK's memory
  allocator after asking for a zero-sized `adsp_shared_reserved` block.
- Cause: that step sums `adsp-rsv-*` properties of the `mediatek,scp-audio-mbox`
  node and always allocates the sum; with no such node the sum is 0, and the
  allocator panics on size 0.
- Fix: `scp-audio-mbox` node with the stock sizes. At the same time the
  other fixup steps were checked for similar traps (no
  `/bootprof`, no zero `sspm-res-ram-size`, 8-byte `kaslr-seed`), and the
  checker was extended accordingly.

## Attempt 3: kernel runs, screen frozen

- LK accepted the images and printed "booting linux"; the kernel came up, the
  initramfs started the USB gadget, and SSH over USB worked.
- simpledrm created `fb0` on the LK framebuffer at `0xfe04f000` (1440x1440,
  stride 5760, a8r8g8b8), but the screen kept showing the LK image.
- Cause: the A61QWERTY46 panel runs in **DSI command mode**. It has its own
  frame memory and only shows a new frame when the display pipeline is
  triggered. Manually writing the mutex enable and DSI start registers showed
  the framebuffer contents with correct colours. The "orange state" warning
  is a second overlay layer that LK leaves enabled.
- Other findings from the kernel log, all fixed or explained for attempt 4:

  | message | cause | action |
  |---|---|---|
  | "KASLR disabled due to lack of seed" | LK fills `kaslr-seed` only if it exists | add `kaslr-seed = /bits/ 64 <0>` |
  | `console=tty0 loglevel=7` arrived as `l=7` | LK reads the v4 command line at the v0-v2 offset, losing 20 bytes | prefix 20 spaces |
  | "Initramfs unpacking failed: invalid magic" | gap reserved for the RECOVERY fragment before bootconfig | harmless, documented |
  | tphy "Failed to create device link" | cyclic fw_devlink between the USB PHY graph port and the controller | warning only, both probe |

## Attempt 4: first usable boot

Configuration: full Titan 2 DTB with SPMI, the MT6363/MT6369 PMICs, input
devices, the display trigger and USB device mode; `clk_ignore_unused
pd_ignore_unused` on the command line.

Works:

- **Display**: simpledrm + fbcon on the LK framebuffer; `mtk-cmdfb-kick`
  verified the pipeline state left by LK, switched off the orange-state layer
  and triggers a frame after each framebuffer update. Console text updates
  live.
- **Keyboard**: AW9523B matrix (`TitanKey`, i2c-6 0x58) via the new
  `aw9523-keypad` driver; letters, Shift, Alt, Sym, Fn, Back, App switch, arrows
  produced events.
- **Touchscreen**: Synaptics RMI4 (i2c-0 0x67, product "57695-006", F12) via the
  in-tree `rmi4_i2c`; supply `mt6369_vtp` raised from 1.8 V to 2.8 V by the
  kernel.
- **Touchpad**: Synaptics S1403 (i2c-2 0x20, RMI4 F11) with GPIO reset and a
  GPIO-switched supply.
- **Volume Up / Volume Down** and **Func2** via gpio-keys.
- **UFS**: three LUNs enumerated, partitions visible, all forced read-only by
  the initramfs.
- **USB**: device mode (mtu3, high speed), NCM + ACM gadget, DHCP to the PC,
  SSH.
- **PMICs**: SPMI bus, MT6363 and MT6369 cores and regulators registered;
  described rails kept always-on, no voltage changed except the touchscreen
  supply.
- Kernel at EL1, KASLR enabled, 12 GiB RAM, pstore available.

Does not work:

- **Power key and Func1** (both are MT6363 PMIC keys): no interrupts. The SPMI
  probe warned while installing the chained handler for the PMIC interrupt
  line (EINT 212). Cause found afterwards: the MT6878 pinctrl EINT table is
  sparse (EINT 0-116 and 196-215), but the EINT core indexes it by EINT number;
  EINT 212 therefore had no interrupt chip. The fix fills the gap 117-195 with
  invalid entries so that index equals EINT number. Pending test on hardware.
- **Reading `/sys/kernel/debug/clk/clk_summary` hangs the bus.** The summary
  reads the state of every registered clock, including providers in blocks
  that may be unpowered (GPU PLLs and the SCP clock block). These providers
  have no consumers yet and are disabled in the DTB for the next build; which
  one caused the hang is not proven.

Not yet enabled (described but disabled, or not described): SCPSYS power
domains, multimedia clock controllers, MT6375 charger, SH366100 fuel gauge,
MT6319 buck, RTC, audio, modem, Wi-Fi/Bluetooth, camera, GPU.

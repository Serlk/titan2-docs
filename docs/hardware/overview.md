<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Unihertz Titan 2: hardware overview

Facts collected from the device tree the stock bootloader passes to Android,
from observing the stock system, and from our own mainline kernel logs.

## SoC and memory

| item | value |
|---|---|
| SoC | MediaTek MT6878 (Dimensity 7300): 4x Cortex-A78 + 4x Cortex-A55 |
| GPU | Mali-G615 (not used yet) |
| DRAM | 12 GiB at `0x40000000`-`0x33fffffff` |
| storage | UFS 3.1, three LUNs (host `0x112b0000`, M-PHY `0x112a0000`) |
| debug UART | UART0 at `0x11001000` (8250, 921600 baud); no accessible connector |
| exception level at kernel entry | EL1 (a hypervisor runs above Linux) |

Selected memory carveouts set up by the boot chain (they appear in
`/reserved-memory` as written by LK; see docs/lk-boot-requirements.md section 9):

| carveout | address | size |
|---|---|---|
| ramoops / pstore | `0x48090000` | 0xe0000 |
| ATF (BL31) | `0x48800000` | 2 MiB |
| TEE | `0x70000000` | 60 MiB |
| SCP firmware | `0xb8000000` | 35 MiB |
| main framebuffer | `0xfe04f000` | 32448 KiB |

## Buses and devices

| bus | controller | devices |
|---|---|---|
| i2c-0 | `0x11c20000` (stock 400 kHz) | touchscreen, Synaptics RMI4 @ 0x67 |
| i2c-2 | `0x11c22000` (100 kHz) | keyboard touchpad, Synaptics S1403 RMI4 @ 0x20 (second-source Hynitron CST76xx @ 0x5b on other batches); rear-display touch @ 0x5a |
| i2c-5 | `0x11280000` (stock 1 MHz) | MT6375 SubPMIC (charger/Type-C/ADC) @ 0x34, 0x4e, 0x1a, 0x4a, 0x64, 0x3f |
| i2c-6 | `0x11e01000` (100 kHz) | AW9523B keyboard matrix @ 0x58; also a buck-boost (rt6160) and the LCD bias |
| i2c-11 | `0x11d71000` | SH366100 fuel gauge @ 0x55, SC854x charge pump @ 0x6f |
| SPMI | PMIF `0x1cc04000`, master `0x1cc01000` | MT6363 (SID 4), MT6369 (SID 5), MT6319 (SID 6), MT6685 RTC/clock (SID 9) |
| USB | SSUSB `0x11201000` (MAC), IPPC `0x11203e00`, T-PHY `0x11f40000` | USB 2.0 high-speed device/host |

## Main display

| item | value |
|---|---|
| panel | A61QWERTY46, 1440x1440, about 66 x 66 mm, about 59.9 Hz |
| interface | MIPI DSI, **command mode** (panel has its own frame memory) |
| framebuffer left by LK | `0xfe04f000`, 1440x1440, stride 5760, `a8r8g8b8` |

Details: hardware/display.md.

## Input

| device | connection | details |
|---|---|---|
| QWERTY keyboard | AW9523B on i2c-6, INT on GPIO12 | hardware/keyboard-aw9523b.md |
| keyboard touchpad | RMI4 on i2c-2, IRQ GPIO75 | hardware/touch-rmi4.md |
| touchscreen | RMI4 on i2c-0, IRQ GPIO7 | hardware/touch-rmi4.md |
| Power, Func1 | MT6363 PMIC keys | hardware/buttons.md |
| Volume Up/Down, Func2 | GPIO54, GPIO55, GPIO73 | hardware/buttons.md |

## Power

| part | role | details |
|---|---|---|
| MT6363 | main PMIC: CPU/SRAM bucks, LDOs, power and Func1 keys | hardware/pmic.md |
| MT6369 | second PMIC: SRAM core, USB 3.3 V, keyboard and touch supplies | hardware/pmic.md |
| MT6319 | buck converter: ISP/CAM vcore, GPU, SoC vcore | hardware/pmic.md |
| MT6375 | SubPMIC: switching charger, BC1.2, ADC, Type-C/PD controller | hardware/charger-mt6375.md |
| SH366100 | fuel gauge (bq27z561-compatible protocol) | hardware/fuel-gauge-sh366100.md |
| battery | Li-ion, full charge capacity about 4600 mAh, stock charge voltage 4.45 V | hardware/charger-mt6375.md |

SoC power domains and clocks: hardware/power-domains.md, hardware/clocks.md.

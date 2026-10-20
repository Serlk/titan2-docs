<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Charger: MT6375 SubPMIC

Status: drivers written (MFD core and charger), DT nodes present but
**disabled**; not yet tested on hardware.

## Connection

| item | value |
|---|---|
| bus | i2c-5 (`0x11280000`); stock runs it at 1 MHz |
| I2C clients | PMU 0x34, TCPC 0x4e, BM (gauge) 0x1a, HK1 0x4a, HK2 0x64, TM 0x3f |
| register addressing | 16-bit: high byte = bank (selects the I2C client), low byte = register |
| interrupt | GPIO15 (EINT 15), active low |
| chip id | 0x100 family, vendor id 0x7 in the upper nibble, revision 2 on this board |

Functions: switching charger with power path, BC1.2 detection, ADC, its own
coulomb counter (BM bank), auxiliary ADC, and a Type-C/PD port controller.

## Charger registers

| reg | field | range |
|---|---|---|
| 0x120 | TOP1: CHG_EN bit 0, BUCK_EN bit 1, OTG bit 2, HZ bit 3 | |
| 0x122 | AICR (input current limit) | 100-3225 mA, 25 mA steps |
| 0x123 | MIVR (minimum input voltage) | |
| 0x125 | CV (charge voltage), bits 6:0 | 3900-4710 mV, 10 mV steps |
| 0x126 | ICHG (charge current) | 300-3150 mA, 50 mA steps |
| 0x127 | safety timer | |
| 0x128 | end-of-charge current (IEOC) | |
| 0x12a | watchdog: 8/40/80/160 s, kick bit 2, enable bit 3 | |
| 0x133 | IR compensation | |
| 0x134 | status | |
| 0x150 / 0x151 | BC1.2 control / status | |
| 0x1d0-0x1df | interrupt events, write 1 to clear | |
| 0x1e0 | status 0 | |
| 0x1f0-0x1ff | interrupt masks | |
| 0x19c, 0x1a4-0x1aa | ADC: one-shot, channel `<< 4` in 0x1a6, big-endian 16-bit result at 0x1aa + 2 * (channel - 1) | 16 channels (VBUS, VSYS, VBAT, IBUS, IBAT, die temperature, D+/D-, SBU, CC, TS, PD VBUS) |

The register map is entirely different from the MT6370 supported in mainline,
so new drivers were written in the same style rather than extending mt6370.
The Type-C controller is standard TCPCI (0x00-0x7f) plus vendor registers
0x80-0xda and an extra bank, close to the mainline `tcpci_mt6370`.

## What the boot chain leaves

| stage | charger actions |
|---|---|
| preloader | detects the chip, battery protection check, runs BC1.2 (an SDP was detected on a PC port), records the port type |
| second-stage loader | starts the charger with board limits (USB 1.2 A, AC 2.05 A / input 3.2 A, non-standard 0.5 A, CDP 1.5 A, temperature thresholds 45/50 C) |
| LK | CV 4350 mV, AICR 500 mA with the ILIM pin disabled, MIVR 4500 mV, ICHG 500 mA |

Expected at kernel entry: charging enabled in hardware at CV 4.35 V, 0.5 A,
input limit 0.5 A. Whether the charger watchdog is enabled is unknown; our
driver prints the full state at probe.

Without any driver the MT6375 keeps charging autonomously with these settings
(CV/CC regulation, termination, safety timer, thermal regulation, input and
battery over-voltage protection). It does **not** watch the battery NTC; in
stock the temperature window is enforced in software from the fuel gauge
reading.

## Stock limits (reference)

| parameter | stock value |
|---|---|
| charge voltage | 4.45 V (runtime CV field 0x37) |
| charge current | 1.2 A on SDP, 2.05 A on DCP |
| input current | up to 3.2 A |
| MIVR | 4.6 V |
| termination | 300 mA, enabled |
| temperature window | 0-50 C (resume at +6 / -3 C) |
| cold limit | below 15 C on DCP: input 1.1 A, charge 1 A |
| watchdog | not enabled by the kernel |

## Mainline driver behaviour

- At probe it first reads and prints the state left by the boot loader.
- Without a `monitored-battery`, it only reports and never writes.
- With a battery: charging off first, watchdog off, IR compensation 0, CV from
  `constant-charge-voltage-max-microvolt`, ICHG from
  `constant-charge-current-max-microamp`, AICR 500 mA, ILIM pin off, IEOC from
  `charge-term-current-microamp`, termination on, 10 h safety timer on.
  BUCK_EN, HZ, OTG and BATFET are left untouched.
- Charging is enabled only by a monitor (every 10 s and on power-ready, detach,
  VBUS over-voltage, timeout and battery over-voltage interrupts) when the
  battery temperature is readable and inside `operating-range-celsius`, and the
  battery voltage is at most CV + 50 mV. Any read error disables charging.
- BC1.2 is not re-run (it would disturb the USB gadget); the port type is taken
  from the preloader's result.
- First-stage DT limits: 4.35 V, 1 A, termination 300 mA, 0-50 C.

Not yet implemented: ADC, BC1.2, Type-C/PD (TCPM), fast charging through the
SC854x charge pump.

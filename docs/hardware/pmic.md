<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# PMICs: MT6363, MT6369, MT6319 (and MT6685)

All four sit on the SPMI bus (hardware/spmi.md). The MT6375 SubPMIC, which is
on I2C, is described in hardware/charger-mt6375.md.

## MT6363 (SID 4): main PMIC

| item | value |
|---|---|
| regulator register block | starts at 0x30: the regulator node has `reg = <0x30>`, and the absolute register address = 0x30 + the offset used by the mainline MT6363 regulator driver (checked for all 140 registers) |
| interrupt groups | BUCK, LDO, PSC, MISC, HK (housekeeping), and AUD; group CON/STATUS registers spaced by 3 |
| interrupt done register | `RCS_INT_DONE`, 0x41b |
| over-current interrupt numbers | VS2 = 0 ... VM18 = 40 |
| key interrupt numbers | 48-53 (press and release interrupts per key) |
| keys | power, home (Func1), home2 |

### Key status bits

Key state is read from the key status register at 0x1e; a cleared bit means
pressed:

| key | bit | mask |
|---|---|---|
| power | 1 | 0x2 |
| (charger detect) | 2 | 0x4 |
| home (Func1, scan code 249) | 3 | 0x8 |
| home2 | 4 | 0x10 |

This layout is the same as on MT6359. The long-press reset configuration is
set by LK and is not changed by Linux.

### Rails used in our DT

Only rails that the stock kernel keeps always on and the SoC cannot run without
are described, as always-on and **without** voltage limits, so Linux never
changes their voltage or mode:

| rail | purpose |
|---|---|
| vbuck5, vbuck6 | CPU clusters |
| vs1, vs2, vs3 | input rails of the LDO groups |
| vio18, vm18 | 1.8 V I/O and memory |
| vsram-cpub, vsram-cpum, vsram-cpul | CPU SRAM |
| vufs18 | 1.8 V UFS I/O |

Rails left out of the DT are registered without constraints and never
touched; this includes the UFS supplies vemc and vufs12, which stock switches.

Voltages read back in boot attempt 4 (all as left by the boot chain): vbuck5
0.8 V, vbuck6 0.612 V, vs1 2.0 V, vs2 1.45 V, vs3 1.05 V, vio18 1.82 V, vm18
1.8 V, vsram-cpub/cpum 0.75 V, vsram-cpul 0.8 V, vufs18 1.86 V, vufs12 1.2 V,
vemc 1.25 V.

### AUXADC

The MT6363 AUXADC is register compatible with the support already present in
the mainline mt6359 AUXADC driver; not described yet.

## MT6369 (SID 5): second PMIC

| item | value |
|---|---|
| regulators | 2 bucks, 15 LDOs |
| VOCAL (calibration) fields | not touched |

| rail | consumer | notes |
|---|---|---|
| vsram-core | SoC SRAM | always-on |
| vusb | USB PHY 3.3 V | always-on, 3.0 V reported |
| vio28 | AW9523B keyboard expander | stock only switches it on, never sets a voltage; 2.8 V observed |
| vtp | touchscreen | stock sets exactly 2.8 V before enabling; comes up at 1.8 V |

## MT6319 (SID 6): buck converter

MT6315-family part. Stock rails: vbuck1 = ISP/camera vcore, vbuck2 = GPU,
vbuck4 = SoC vcore. Disabled in our DT for now: the mainline MT6315 driver
assumes a multi-phase vbuck1 for SID 6 (phase mask 0xb; stock uses 0x1) and arms
a power-off sequence on shutdown, which the factory kernel does not do. The ISP
and camera power domains need vbuck1 as their supply.

## MT6685 (SID 9): RTC and clock buffer

Provides the RTC and the 32 kHz clock. Same RTC IP family as MT6397 (register
base 0x580). Not supported yet; its probe writes RTC control registers, so it
needs a separate review before enabling.

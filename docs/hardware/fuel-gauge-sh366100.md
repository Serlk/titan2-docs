<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Fuel gauge: Sinowealth SH366100

Status: supported through the mainline `bq27xxx` driver (new chip entry,
read-only); DT node present but **disabled** until tested.

## Connection

| item | value |
|---|---|
| bus | i2c-11 (`0x11d71000`) |
| address | 0x55 |
| interrupt | none (polled) |
| neighbour | SC854x charge pump at 0x6f |

## Protocol

The SH366100 is protocol compatible with the TI bq27z561 / bq28z610 family:
standard 16-bit little-endian commands, ManufacturerAccess through 0x3e/0x40,
block checksum/length at 0x60/0x61, the TI default unseal and full-access keys,
and SEAL as MAC 0x0020.

### Standard commands

| cmd | quantity | format |
|---|---|---|
| 0x06 | Temperature (battery NTC) | u16, 0.1 K |
| 0x08 | Voltage | u16, mV |
| 0x0a | BatteryStatus | flags |
| 0x0c | Current (instantaneous) | s16, mA, positive = charging |
| 0x10 | RemainingCapacity | mAh |
| 0x12 | FullChargeCapacity | mAh |
| 0x14 | AverageCurrent | s16, mA |
| 0x28 | InternalTemperature | u16, 0.1 K |
| 0x2c | RelativeStateOfCharge | % |
| 0x2e | StateOfHealth | % |
| 0x3c | DesignCapacity | mAh |

Example reading from the stock system while charging from a PC port: 97 %,
4418 mV, +448 mA, 23.2 C, full charge capacity 4624 mAh, remaining 4447 mAh,
state of health 97 %.

### Gauge MAC commands (for reference)

The factory system uses MAC commands to read version and configuration
information, to enable gauging (0x0021) and lifetime data (0x0067), to reset
(0x0041), and to update the battery profile and the gauge firmware. The profile
is stored in the gauge itself, so none of this is needed by Linux.

## Mainline

- `bq27xxx_battery` with a new `SH366100` chip: bq27z561 register layout plus
  InternalTemperature at 0x28, no data-memory registers. The driver therefore
  **never writes** to the gauge (no unseal, no data flash, no reset).
- `current_now` reports AverageCurrent (0x14), as bq27xxx does for all
  z-series chips.
- Unverified: the BatteryStatus bit layout (assumed as on bq27z561: FD bit 4,
  FC bit 5, DSG bit 6) and the time-to-empty/full and cycle-count commands.

Device tree:

```
&i2c11 {
	fuel-gauge@55 {
		compatible = "sinowealth,sh366100";
		reg = <0x55>;
	};
};
```

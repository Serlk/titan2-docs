<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# SPMI bus (PMIF) on MT6878

## Controller

| item | value |
|---|---|
| PMIF registers | `0x1cc04000`, size 0x900 |
| SPMI master registers | `0x1cc01000`, size 0x100 |
| buses | one SPMI bus |
| register layout | same as MT8195 PMIF for all shared fields; SPMI master same as MT8195/MT6853 |
| SoC channel | 2 |
| SPMI version | 2 |
| clocks | none taken by the driver; the PMIF/SPMI clocks are always on and are left alone |
| interrupt used | RCS ("request capable slave") line: **EINT 212** through the pin controller, level high |

The stock tree also lists the PMIF interrupt (GIC SPI 496) and a NACK interrupt
(GIC SPI 514); mainline does not use them.

## Slaves

| SID | chip | role |
|---|---|---|
| 4 | MT6363 | main PMIC |
| 5 | MT6369 | second PMIC |
| 6 | MT6319 | buck converter (MT6315 family) |
| 9 | MT6685 | RTC and 32 kHz clock buffer (not supported yet) |

## RCS interrupt handling

PMIC interrupts reach the SoC as RCS requests:

1. A slave raises its RCS request; PMIF signals EINT 212.
2. The handler masks and acknowledges EINT 212, reads the per-slave EINT status
   bytes, and dispatches one interrupt per requesting slave.
3. The per-slave status byte is cleared by writing it back. Slaves that have no
   handler (for example the MT6685 at SID 9) must be cleared too, otherwise the
   RCS line stays asserted.
4. The PMIC's interrupt thread handles its own interrupt groups and then writes
   `RCS_INT_DONE` (MT6363 register 0x41b). The PMIC sends no further RCS request
   before that.

Two bugs were fixed in the mainline PMIF driver while porting:

- The slave id for a status bit was computed as `bit / 7` instead of `bit / 8`;
  for some bits the handler cleared the wrong byte and looped forever in hard
  interrupt context.
- The RCS interrupt chip used an `irq_eoi` callback together with
  `handle_level_irq`, which never calls it; the status byte was never cleared.
  It now uses `irq_ack`.

## Pin controller EINT table

The MT6878 EINT number space goes up to 216, but only EINT 0-116 and 196-215
exist. The EINT core indexes the pin table **by EINT number**. With a packed
table (137 entries) EINT 196-215 ended up at indices 117-136 and EINT 212 had no
interrupt chip, so the SPMI driver could not install its chained handler
(kernel warning in boot attempt 4; power key and Func1 dead). The fix fills
indices 117-195 with invalid entries so that the table has 216 entries and the
index equals the EINT number.

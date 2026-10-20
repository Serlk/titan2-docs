<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# SoC power domains (SCPSYS / SPM)

The MT6878 power controller lives in the SPM block at `0x1c001000`. Most
domains are switched directly through a PWR_CON register; two are controlled
through a hardware voter (HWV, syscon at `0x10320000`). Bus protection steps use
the infracfg_ao (`0x10001000`), VLP config (`0x1c00c000`) and EMI config
(`0x10270000`) blocks.

Status in our tree: the controller and all 22 domains are implemented and their
data were compared field by field against the stock definition with zero
differences, but the controller node is **disabled** in the DT. Linux does not
touch any domain yet; the boot chain leaves audio, mm-infra and disp (for the
framebuffer) and SSUSB on.

## PWR_CON conventions

- Status: bit 30 = power on acknowledged, bit 31 = second power on
  acknowledged (the modem domain uses only bit 30).
- SRAM: power-down bits and acknowledge bits as listed; "clear-to-on" unless
  marked inverted.
- Bus protection: set/clear/status register triplets in the named block with
  the listed mask.

## Domains

Offsets are relative to the SPM base unless stated otherwise.

| id | domain | control | SRAM pdn / ack | bus protection (block: set/clr/sta, mask) | flags / notes |
|---|---|---|---|---|---|
| 0 | md (modem) | 0xe00 | - | infra c54/c58/c5c 0x200; infra c44/c48/c4c 0x800; emi 84/88/8c 0xc0 (with ack) | external buck isolation 0xf24 bits 1:0; modem power sequence; no reset_b; default off. Not described in the DT |
| 1 | conn | 0xe04 | - | infra c94.. 0x2; infra c54.. 0x1000; infra c94.. 0x1; infra c44.. 0x100 | default off |
| 2 | audio | 0xe18 | 0x100 / 0x1000 | infra c84.. 0x40 | clock: audio |
| 3 | csi-rx | 0xe9c | - | - | |
| 4 | ssrsys | HWV bit 1 | - | - | secure infra power request |
| 5 | ssusb | 0xea8 | 0x100 / 0x1000 | infra c84.. 0x80 | |
| 6 | mfg0-shutdown | 0xeb4 | 0x100 / 0x1000 | infra ca4.. 0x10; infra c44.. 0x200 | SRAM isolation; default off; GPU power is handled by its microcontroller |
| 7 | mm-infra | HWV bit 0 | - | - | clock: mm_infra; parent of disp, ven0, vde0, mm-proc-dormant (and camera/ISP) |
| 8 | disp | 0xe70 | 0x100 / 0x1000 | infra c14.. 0x40003 | parent mm-infra; clock: disp |
| 9 | ven0 | 0xe40 | 0x100 / 0x1000 | infra c14.. 0x1000; infra c24.. 0x1000 | parent mm-infra |
| 10 | cam-vcore | 0xe5c | - | infra c14.. 0x80000000; infra c24.. 0x600 | supply MT6319 vbuck1 |
| 11 | cam-ccu | 0xe60 | 0x100 / 0x1000 | infra cc4.. 0x500; infra cc4.. 0x1a00; infra c44.. 0x2000 | |
| 12 | cam-ccu-ao | 0xe64 | - | - | |
| 13 | cam-main | 0xe48 | 0x100 / 0x1000 | infra c14.. 0x20000000; 0x40000000 | |
| 14 | cam-suba | 0xe50 | 0x100 / 0x1000 | infra c14.. 0x2000000; 0x4000000 | |
| 15 | cam-subb | 0xe54 | 0x100 / 0x1000 | infra c14.. 0x8000000; 0x10000000 | |
| 16 | isp-vcore | 0xe34 | - | infra c14.. 0x100; infra c24.. 0x180 | supply MT6319 vbuck1 |
| 17 | isp-main | 0xe28 | 0x100 / 0x1000 | infra c14.. 0x4, 0x8, 0x10, 0x20 | |
| 18 | isp-dip1 | 0xe2c | 0x100 / 0x1000 | infra c14.. 0x40, 0x80 | |
| 19 | vde0 | 0xe38 | 0x100 / 0x1000 | infra c14.. 0x100000; infra c24.. 0x2000 | parent mm-infra |
| 20 | mm-proc-dormant | 0xe7c | 0x200 / 0x2000, set-to-on | vlp 214/218/220 0x100; 0x600 | SRAM isolation, inverted SRAM pdn; parent mm-infra; clock: mmup |
| 21 | apu | - | - | - | controlled through the APU subsystem; not supported |

"infra c14.." means the triplet c14/c18/c1c (set/clear/status), and so on.

HWV registers (offsets in the HWV block): set 0x198, clear 0x19c, done 0x141c,
enable 0x1410, set status 0x146c, clear status 0x1470.

## Mainline notes

- Two secure infra domains (mm-infra and ssrsys) are requested through a
  firmware call; requests are now reference counted, as stock does.
- The controller mixes HWV domains and direct domains in one index space,
  because mm-infra (HWV) is the parent of direct domains.
- Differences in sequencing kept from mainline (same as MT8196): both power-on
  bits set together, a single bus protection write instead of stock's retries,
  slightly different SRAM isolation ordering.
- Sub-system clock gates (mmsys, vdec, venc, cam, img) that stock enables before
  releasing bus protection do not exist in mainline yet.
- Do not enable the controller without care: probe powers on every domain not
  marked default-off, and on any probe error it powers down the domains already
  registered, **including disp**, which blanks the screen. Enable it together
  with `power-domains = <&spm MT6878_POWER_DOMAIN_DISP>` on the
  simple-framebuffer node.

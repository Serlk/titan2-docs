<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Clocks: summary

## Controllers

| block | address | mainline compatible | state in our DT |
|---|---|---|---|
| topckgen | `0x10000000` | `mediatek,mt6878-topckgen` | enabled |
| infracfg_ao | `0x10001000` | `mediatek,mt6878-infracfg-ao` | enabled |
| apmixedsys (PLLs) | `0x1000c000` | `mediatek,mt6878-apmixedsys` | enabled |
| pericfg_ao | `0x11036000` | `mediatek,mt6878-pericfg-ao` | enabled |
| vlpckgen | `0x1c012000` | `mediatek,mt6878-vlpckgen` | enabled |
| GPU PLL / GPU SC PLL | `0x13fa0000` / `0x13fa0c00` | `mediatek,mt6878-mfgpll-pll-ctrl`, `-mfgscpll-pll-ctrl` | to be disabled (see below) |
| SCP clock config | `0x1cb21000` | `mediatek,mt6878-scp-cfg` | to be disabled (see below) |
| mmsys0 | `0x14000000` | `mediatek,mt6878-mmsys` | disabled |
| imgsys and its sub-blocks | `0x15000000`-`0x15780000` | `mediatek,mt6878-imgsys*` | disabled |
| vdecsys / vencsys | `0x1602f000` / `0x17000000` | `mediatek,mt6878-vdecsys`, `-vencsys` | disabled |
| camsys blocks, ccusys, cam vcore | `0x1a000000`-`0x1b204000` | `mediatek,mt6878-camsys-*`, `-ccusys` | disabled |
| mminfra config | `0x1e800000` | `mediatek,mt6878-mminfra-config` (uses the hardware voter) | disabled |
| mdpsys | `0x1f000000` | `mediatek,mt6878-mdpsys` | disabled |
| audiosys | `0x11050000` | not ported (belongs with the audio front-end driver) | - |

Multimedia controllers stay disabled until they get power domains: reading
their registers while the domain is off hangs the bus.

## Comparison with stock

444 clocks (muxes, gates, dividers, PLLs) were extracted from both the stock
definitions and the mainline drivers and compared field by field (parents in
order, mux/set/clear registers, shift/width, gate bits, update bits, PLL enable
bits): no differences remain except intentional ones. Clock IDs are identical
to the stock numbering in every block, so stock DT clock references carry over
unchanged.

Fixes made to the existing mainline MT6878 support:

- **PLL enable**: the PLL enable bit argument was dropped, so every PLL wrote
  bit 0 of the common enable register; the PLLs actually use a shared
  PLLEN_ALL register with set/clear aliases (0x70/0x74/0x78), bits 9..0 =
  armpll_ll, armpll_bl, ccipll, mainpll, univpll, msdcpll, mmpll, ufspll,
  apll1, apll2.
- **topckgen table overflow**: clock IDs went up to 274 while only 257 slots
  were allocated; 17 entries were written past the end of the array. The
  table is now sized by the highest ID.
- Missing clocks added: `disp_pwm_sel` (with two peripheral gates for the
  backlight PWM, a vendor addition), two GPU mux inputs, the audio composite
  clocks and seven audio divider gates.

Intentional differences: univpll and mmpll are marked always-on (stock controls
them through the hardware voter and a reset sequence, not implemented); a few
multimedia muxes that stock gates through the voter or the MMDVFS firmware are
plain muxes without a gate.

## Boot safety

- No `assigned-clocks` anywhere; the kernel does not change rates or parents.
- Critical clocks (CPU PLLs, CCI PLL, main PLL, univpll, mmpll, SCP mux) are
  only "enabled" by writing their set bit, which is idempotent.
- The kernel runs with `clk_ignore_unused` for now.

## Known problem: clk_summary hangs the bus

Reading `/sys/kernel/debug/clk/clk_summary` queries the enable/prepare state of
every registered clock and so reads registers of every provider. Providers in
blocks that may be unpowered are the GPU PLL controllers (powered by the GPU
microcontroller) and the SCP clock block. None of them has a consumer yet, so
they are being disabled in the DT. Which of the three caused the hang in boot
attempt 4 is not proven.

<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Main display: DSI command mode and frame triggering

## Panel and pipeline state left by LK

The main panel (A61QWERTY46, 1440x1440) is driven in **MIPI DSI command mode**:
it keeps its own frame memory and shows a new frame only when the display
pipeline sends one. LK initialises the panel and the pipeline and leaves a
static image. Writing to the framebuffer alone does not change the screen.

LK leaves the pipeline as follows (values read on the device):

| block | base | register (offset) | value | meaning |
|---|---|---|---|---|
| DSI0 | `0x1401a000` | START (+0x00) | 1 | last frame start |
| | | INTSTA (+0x0c) | bit 31 | busy while a frame is being sent |
| | | MODE_CTRL (+0x14) | 0 | command mode |
| | | CMDQ_SIZE (+0x60) | 0x8001 | one command queue entry |
| | | CMDQ0 (+0xd00) | 0x002c3909 | DCS long write of `write_memory_start` (0x2c) |
| MUTEX | `0x14001000` | MUTEX0 EN (+0x20) | 0 when idle | one-shot: set to 1 per frame, clears itself after the frame |
| | | MUTEX0 MOD (+0x30) | 0x1200ffc7 | modules in the path |
| OVL0 | `0x14002000` | SRC_CON (+0x2c) | 3 | layers 0 and 1 enabled |
| | | L0 address (+0xf40) | `0xfe04f000` | the main framebuffer |
| | | L1 address (+0xf60) | `0xfe838000` | the "orange state" warning overlay |

Layer address registers follow the pattern `0xf40 + 0x20 * n` (verified for
layers 0 and 1).

Framebuffer: `0xfe04f000`, 1440x1440, stride 5760 bytes, `a8r8g8b8`, mapped
write-combined. LK reports it in `/chosen` (`atag,videolfb*`) and as the
`framebuffer` carveout; the carveout needs `no-map` for simpledrm to map it
(docs/lk-boot-requirements.md section 9).

## Sending one frame by hand

```
devmem 0x14001020 1     # MUTEX0 enable (one frame)
devmem 0x1401a000 0     # DSI START 0 -> 1
devmem 0x1401a000 1
devmem 0x1400202c 1     # OVL0: keep only layer 0 (hides the orange-state text)
```

## mtk-cmdfb-kick (our interim driver)

A small driver for our tree only; it depends on the state left by LK and is not
intended for upstream. The proper solution is mediatek-drm support for MT6878.

- Device tree: `compatible = "mediatek,mt6878-cmdmode-fb-kick"`, three register
  ranges named `mutex`, `ovl`, `dsi`, and `mediatek,framebuffer` pointing to the
  simple-framebuffer node. Optional `power-domains` (DISP) and `clocks`.
- simpledrm (drm/sysfb) gains a flush notifier: after every update of the
  scanout buffer (damage-based atomic update, and the clear on disable) it
  notifies subscribers with the device and the damaged rectangle. All writers
  (fbcon via deferred I/O, DRM clients) go through it, so no timer is needed.
- Probe checks the LK state and refuses to touch anything unless it matches:
  DSI in command mode, at least one queued command equal to 0x002c3909, MUTEX0
  module mask non-zero and EN clear, only OVL layers 0..1 enabled and one of
  them pointing at the simple-framebuffer base. It then disables all other OVL
  layers and triggers one frame.
- On each notification: write barrier, then from a work item wait for idle
  (INTSTA bit 31 clear and MUTEX0 EN clear; poll every 500 us, 100 ms timeout),
  set MUTEX0 EN, toggle DSI START 0 -> 1. Requests arriving during a frame are
  merged into the next one. Before every frame the DSI state is re-checked;
  ten consecutive timeouts or a mismatch stop the driver with a register dump.
- Parameter `mtk_cmdfb_kick.refresh_hz` (default 0, at most 120) adds a
  periodic frame for debugging. sysfs counters: `frames`, `timeouts`,
  `stopped`.
- Idle cost: no frames are sent while nothing is drawn.

Observed in boot attempt 4:

```
mtk-cmdfb-kick 14001000.display-trigger: LK state: DSI MODE 0x0 CMDQ_SIZE 0x8001 CMDQ0 0x2c3909; MUTEX0 EN 0x0 MOD 0x1200ffc7; OVL SRC_CON 0x3
mtk-cmdfb-kick 14001000.display-trigger: OVL SRC_CON 0x3 -> 0x1 (only layer 0)
```

## Constraints

- The DISP power domain and the multimedia clocks must stay on: the SCPSYS
  controller is disabled in our DT and the kernel runs with
  `clk_ignore_unused pd_ignore_unused`. Accessing DISP registers with the domain
  off hangs the bus.
- If mediatek-drm for MT6878 appears, this node must be removed; both would
  program the same registers.
- Backlight and panel power are left as LK set them; brightness control is not
  available yet.

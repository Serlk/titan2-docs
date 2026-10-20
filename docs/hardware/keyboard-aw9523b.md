<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# QWERTY keyboard: AW9523B matrix

The keyboard is a key matrix scanned through an Awinic **AW9523B** 16-bit I/O
expander. The expander has no keypad engine; the host scans the matrix by
reprogramming the port directions.

## Connection

| item | value |
|---|---|
| bus | i2c-6 (`0x11e01000`), 100 kHz |
| address | 0x58 |
| interrupt | GPIO12 (EINT 12), INTN of the expander, active low, level |
| reset | GPIO53, low = reset |
| supply | MT6369 vio28 (switched on only; 2.8 V observed) |
| unused | GPIO21 (named as an LED GPIO in the stock DT, not driven) |
| rows (inputs) | P0_0 .. P1_0 = lines 0..8 (mask 0x01ff) |
| columns (outputs) | P1_1 .. P1_7 = lines 9..15 (mask 0xfe00) |

Reset sequence used by stock: reset high, 1 ms, low, 1 ms, high, 6 ms; then read
the ID register until it returns 0x23 (up to 5 tries, 5 ms apart).

## AW9523B registers used

| reg | name | use |
|---|---|---|
| 0x00 / 0x01 | INPUT P0 / P1 | row state; reading clears the interrupt |
| 0x02 / 0x03 | OUTPUT P0 / P1 | 0x0000 (driven columns pull low) |
| 0x04 / 0x05 | CONFIG P0 / P1 | 1 = input, 0 = output |
| 0x06 / 0x07 | INT disable P0 / P1 | 1 = interrupt disabled |
| 0x10 | ID | reads 0x23 |
| 0x11 | GCR | bit 4: port 0 push-pull |
| 0x12 / 0x13 | LED/GPIO mode | 1 = GPIO for all matrix lines |

Multi-byte accesses rely on register auto-increment (P0 then P1).

## Initialisation

1. All interrupts disabled (0x06/0x07 = 0xff).
2. All 16 lines in GPIO mode.
3. Rows as inputs, columns as outputs; GCR bit 4 set; outputs low.
4. Row interrupts enabled; INPUT read once to clear a pending interrupt.

## Scanning

- Idle: all columns driven low, row interrupts enabled. A key press pulls a row
  low and asserts INTN.
- On interrupt: mask row interrupts, wait a short debounce (1 ms), then scan:
  for each column, configure only that column as an output (CONFIG =
  `0xffff & ~BIT(column)`) and read INPUT (2 bytes). A cleared row bit means
  the key at (row, column) is pressed.
- While any key is held, rescan periodically (our driver: every 5 ms). After
  all keys are released, restore idle and re-enable row interrupts, then
  re-check the rows once to catch a press that raced the re-arm.
- Cost at 100 kHz: two transfers per column, about 7 ms per scan with our
  driver (stock uses three transfers per column, about 11 ms).

## Keymap

Matrix position (row, column) to Linux key code, as the stock firmware maps it.
Positions marked "?" have a code in the stock table but probably no physical
key (duplicates or codes never observed).

| row \ col | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|---|
| 0 | Y | T | G | H | Space | B | F1 ? |
| 1 | U | R | F | J | V | N | U ? |
| 2 | I | E | D | K | C | M | K ? |
| 3 | O | W | S | L | X | Enter | F ? |
| 4 | P | Q | A | Backspace | Z | Z ? | X ? |
| 5 | Left Shift | Sym (253) | Back | App switch (580) | F10 ? | N ? | M ? |
| 6 | Fn (251) | Down | Up | Enter ? | F11 ? | F13 ? | AltErase ? |
| 7 | . (dot) | Left | Right | Space ? | Volume Up ? | Stop ? | Again ? |
| 8 | Right Alt | Undo ? | Front ? | Copy ? | Open ? | Paste ? | Find ? |

Notes:

- Codes 251 (Fn) and 253 (Sym) are vendor codes without a standard Linux name.
  580 is `KEY_APPSELECT`, 158 `KEY_BACK`.
- Row 8 (P1_0) carries only the Alt key. The stock firmware uses this to let
  Alt wake the screen: with the screen off it reconfigures the expander so
  that only P1_0 raises an interrupt and reports `KEY_POWER` on a change.
- Keys pressed during the short input test of boot attempt 4 all produced the
  expected codes: Q, W, E, R, U, I, O, S, D, F, G, H, J, K, V, Space, Shift,
  Alt, Sym, Fn, Back, App switch, Up and Down. The remaining positions have not
  been exercised on mainline yet.

## Mainline driver: aw9523-keypad

- Compatible `awinic,aw9523b-keypad`; properties: `interrupts`, `reset-gpios`,
  `vio-supply`, `awinic,row-lines` (0..8), `awinic,column-lines` (9..15),
  `linux,keymap` (`MATRIX_KEY(row, column, code)`), `debounce-delay-ms`,
  `poll-interval`, `label` (input device name, "TitanKey"), `wakeup-source`,
  optional autorepeat.
- Reports `EV_KEY` plus `MSC_SCAN`.
- With `wakeup-source` the expander stays armed during system suspend and any
  key wakes the system. `KEY_POWER` is not substituted; that is an Android
  screen-off policy, not a hardware feature.
- Not implemented: the stock single-key screen-off mode, ghost-key filtering,
  the vendor sysfs files, and the coupling to the second-source touchpad
  driver.

The generic `pinctrl-aw9523` + `gpio-matrix-keypad` combination also works but
needs about 80 single-pin reads and 30 writes per scan (10-15 ms at 100 kHz),
plus a nested interrupt per pin, which is why a dedicated driver was written.

Open questions: whether the rows and INTN have board pull-ups (stock relies on
the preloader's pin setup), and whether i2c-6 can run at 400 kHz (other
devices share the bus).

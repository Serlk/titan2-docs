<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Side buttons

| button | source | line | polarity | Linux code | mainline driver | status (attempt 4) |
|---|---|---|---|---|---|---|
| Power | MT6363 PMIC key | PMIC interrupt via SPMI RCS (EINT 212) | key status bit 1 cleared = pressed | `KEY_POWER` (116) | mtk-pmic-keys | not working (see hardware/spmi.md) |
| Func1 (programmable) | MT6363 PMIC "home" key | as above | bit 3 cleared = pressed | 249 | mtk-pmic-keys | not working |
| Func2 (programmable) | GPIO73 | EINT 73, both edges | low = pressed | 250 | gpio-keys | works |
| Volume Up | GPIO54 | EINT 54 | low = pressed, pull-up | `KEY_VOLUMEUP` (115) | gpio-keys | works |
| Volume Down | GPIO55 | EINT 55 | low = pressed, pull-up | `KEY_VOLUMEDOWN` (114) | gpio-keys | works |

Notes:

- The GPL v2 module gpio_key.ko of Titan 2 V01.00.14 uses a 200 us EINT
  debounce for Func2; our DT uses a 10 ms gpio-keys debounce. Volume keys use 30 ms.
- The MT6363 also has a third key input ("home2", status bit 4) that stock
  maps to 250; on the Titan 2 code 250 comes from GPIO73 instead, so home2 is
  not described.
- In the stock Android key layout 249 and 250 are remapped by the framework to
  vendor-specific programmable functions; mainline just reports the raw codes.
- LK boot key combinations: Volume Up + Power during boot opens the boot menu;
  Volume Up alone selects recovery (docs/lk-boot-requirements.md section 12).

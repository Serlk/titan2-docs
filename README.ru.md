<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# titan2-docs

[English version](README.md)

Документация к **альфа-версии mainline Linux 7.2 для Unihertz Titan 2**
(MediaTek MT6878 / Dimensity 7300). Ядро загружает стоковый загрузчик LK
телефона, сам загрузчик не изменяется.

В репозитории только документация: интерфейс загрузки стокового LK, итоги
первых попыток загрузки и описание железа, собранное при запуске mainline-ядра.

## Состояние альфы (4-я загрузка)

Работает: экран (simpledrm на фреймбуфере LK и запуск кадров для DSI-панели в
командном режиме), QWERTY-клавиатура (матрица на AW9523B), тачскрин и тачпад
клавиатуры (Synaptics RMI4), клавиши громкости, Func2, UFS (только чтение),
USB в режиме устройства (NCM + ACM) и SSH, регуляторы MT6363/MT6369.

Пока не работает: клавиша питания и Func1 (прерывание PMIC, исправление ждёт
проверки), чтение `clk_summary` вешает шину; зарядка, датчик заряда, звук,
модем, Wi-Fi, камера и GPU не включены.

## Документы

Загрузка:

- [docs/lk-boot-requirements.md](docs/lk-boot-requirements.md) — что стоковый
  LK требует от `boot`, `vendor_boot`, `init_boot`, `dtbo`, `vbmeta` и DTB
  (версия заголовка, адреса загрузки, ошибка смещения cmdline, фрагменты
  рамдиска, обязательные и запрещённые узлы, узлы для загрузчиков прошивок,
  слоты, восстановление)
- [docs/boot-attempts.md](docs/boot-attempts.md) — попытки загрузки 1–4: что
  пробовали, что произошло, что помогло
- [docs/DUALBOOT.md](docs/DUALBOOT.md) — Android и mainline Linux рядом: меню
  titan2-boot, раздел `linux`, защита разделов Android, установка (на
  английском); [docs/DUALBOOT-ROLLBACK.md](docs/DUALBOOT-ROLLBACK.md) — как
  всё вернуть

Железо:

- [docs/hardware/overview.md](docs/hardware/overview.md) — SoC, память, шины,
  карта устройств
- [docs/hardware/display.md](docs/hardware/display.md) — DSI-панель в командном
  режиме, регистры конвейера, запуск кадров
- [docs/hardware/keyboard-aw9523b.md](docs/hardware/keyboard-aw9523b.md) —
  матрица клавиатуры, сканирование, раскладка
- [docs/hardware/touch-rmi4.md](docs/hardware/touch-rmi4.md) — тачскрин и
  тачпад клавиатуры (параметры RMI4)
- [docs/hardware/buttons.md](docs/hardware/buttons.md) — питание, громкость,
  Func1, Func2
- [docs/hardware/spmi.md](docs/hardware/spmi.md) — контроллер SPMI,
  RCS-прерывания, таблица EINT
- [docs/hardware/pmic.md](docs/hardware/pmic.md) — MT6363, MT6369, MT6319,
  MT6685
- [docs/hardware/charger-mt6375.md](docs/hardware/charger-mt6375.md) — зарядник
  MT6375
- [docs/hardware/fuel-gauge-sh366100.md](docs/hardware/fuel-gauge-sh366100.md) —
  датчик заряда SH366100
- [docs/hardware/power-domains.md](docs/hardware/power-domains.md) — домены
  питания SCPSYS
- [docs/hardware/clocks.md](docs/hardware/clocks.md) — контроллеры тактов
- [docs/hardware/usb-gadget.md](docs/hardware/usb-gadget.md) — USB в режиме
  устройства и отладочный initramfs

Сами документы написаны по-английски.

## Связанные репозитории

- **titan2-linux** — патчи ядра и device tree (на базе mainline 7.2)
- **titan2-tools** — сборка образов (`scripts/make-mainline-images.sh`),
  проверка DTB на требования LK (`scripts/lk-fdt-check.py`), initramfs и
  вспомогательные скрипты

## Предупреждение

Любая прошивка может сделать телефон незагружаемым. Никогда не прошивайте
`preloader`, `lk`, `tee`, `seccfg` и не трогайте разделы калибровок (`nvram`,
`nvdata`, `nvcfg`, `proinfo`, `protect1/2`). Сделайте полный бэкап и до любых
опытов прочитайте, как вернуться в Android
([docs/lk-boot-requirements.md](docs/lk-boot-requirements.md), разделы 11–12).

## Отказ от ответственности

Это независимый проект сообщества. Он не связан с Unihertz, MediaTek или
другими производителями, не одобрен и не поддерживается ими. Названия
продуктов и товарные знаки принадлежат их владельцам. Документы описывают
наблюдаемое поведение и факты о железе своими словами; кода и бинарных файлов
производителей в них нет. Используйте на свой риск.

## Как делается проект и с кем он связан

- Проект разрабатывается с помощью ИИ-ассистентов (Anthropic Claude). В коммитах стоит строка
  `Co-Authored-By: Claude`; каждое изменение проверяет и испытывает на устройстве человек —
  мейнтейнер (serlk), он же за него отвечает.
- Проект **не связан с postmarketOS** и MT6878-Mainline. Мы опираемся на их GPL-работу над ядром
  (спасибо!), но postmarketOS не принимает вклад, сделанный с помощью ИИ, поэтому отсюда ничего
  не отправляется в postmarketOS, и это не порт postmarketOS.

## Источники

Сведения в этих документах получены с самого телефона: его журналы загрузки,
чтение регистров и шин на устройстве, заводское дерево устройств и собственные
модули ядра телефона, объявленные под GPL, и анализ поведения стокового
загрузчика (его код не воспроизводится). Для сравнения использовались
публичные исходники MediaTek (GPL/BSD) и upstream Linux. Проприетарного кода,
прошивок и бинарных файлов здесь нет.

## Лицензия

GPL-2.0-or-later, см. [LICENSE](LICENSE). Copyright (c) 2026 serlk.

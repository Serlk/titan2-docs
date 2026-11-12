<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# titan2-docs

[English version](README.md)

Документация к **mainline Linux 7.2 для Unihertz Titan 2** (MediaTek MT6878 /
Dimensity 7300): как его собрать, как поставить рядом с Android, что работает и как ведут себя
железо и стоковый загрузчик. Стоковый загрузчик телефона (LK) используется как есть и не
изменяется.

## С чего начать

- [docs/STATUS.ru.md](docs/STATUS.ru.md) — что работает, что частично и чего нет (звонки, SMS,
  мобильный интернет, Wi-Fi, GPU, зарядка, сон, камеры, GNSS, отпечаток, …)
- [docs/BUILD-LINUX.ru.md](docs/BUILD-LINUX.ru.md) — сборка всего из исходников: ядро, DTB,
  initramfs, корневая ФС Alpine, меню загрузки, по желанию Mesa и libcamera
- [docs/INSTALL-LINUX.ru.md](docs/INSTALL-LINUX.ru.md) — установка второй системой рядом с
  Android: бэкапы, меню загрузки, раздел `linux`, первая загрузка, SSH и Wi-Fi, обновление ядра,
  файлы с вашего телефона, модем и звонки, дополнительные части, возврат, что делать при
  проблемах

Коротко (12.11.2026): работают звонки VoLTE, SMS, интернет по LTE, Wi-Fi, экран с GPU, зарядка,
датчики, камеры и GNSS; сон работает, но потребляет около 80 мА; отпечаток пока не работает.
Собственное окружение рабочего стола разрабатывается и будет опубликовано позже.

## Документы

Установка и состояние:

- [docs/STATUS.ru.md](docs/STATUS.ru.md), [docs/BUILD-LINUX.ru.md](docs/BUILD-LINUX.ru.md),
  [docs/INSTALL-LINUX.ru.md](docs/INSTALL-LINUX.ru.md) (по-английски — те же имена без `.ru`)

Загрузка:

- [docs/lk-boot-requirements.md](docs/lk-boot-requirements.md) — что стоковый
  LK требует от `boot`, `vendor_boot`, `init_boot`, `dtbo`, `vbmeta` и DTB
  (версия заголовка, адреса загрузки, ошибка смещения cmdline, фрагменты
  рамдиска, обязательные и запрещённые узлы, узлы для загрузчиков прошивок,
  слоты, восстановление)
- [docs/boot-attempts.md](docs/boot-attempts.md) — попытки загрузки 1–4 первой альфы: что
  пробовали, что произошло, что помогло
- [docs/DUALBOOT.md](docs/DUALBOOT.md) — Android и mainline Linux рядом: меню
  titan2-boot, раздел `linux`, защита разделов Android, установка;
  [docs/DUALBOOT-ROLLBACK.md](docs/DUALBOOT-ROLLBACK.md) — как всё вернуть

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

Документы о загрузке и железе написаны по-английски.

## Репозитории

| репозиторий | что в нём |
|---|---|
| [titan2-linux](https://github.com/Serlk/titan2-linux) | ядро (на базе mainline 7.2) и деревья устройств; ветка `titan2` |
| [titan2-tools](https://github.com/Serlk/titan2-tools) | скрипты сборки, initramfs, сборщик корневой ФС Alpine, службы и программы userspace (серверы модема, звук в звонке, плагин ModemManager, vendor-окружение) |
| [titan2-boot](https://github.com/Serlk/titan2-boot) | меню загрузки (Android / Linux) после стокового LK |
| [titan2-docs](https://github.com/Serlk/titan2-docs) | эта документация |
| [titan2-twrp](https://github.com/Serlk/titan2-twrp) | TWRP для Titan 2 |
| [titan2-camera](https://github.com/Serlk/titan2-camera) | по желанию: драйверы камер и поддержка в libcamera |
| [titan2-gnss](https://github.com/Serlk/titan2-gnss) | по желанию: GNSS и A-GPS |
| [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | по желанию: отпечаток (пока не работает) |
| [titan2-android](https://github.com/Serlk/titan2-android) | crDroid 16 (Android) для Titan 2 |

## Предупреждение

Любая прошивка может сделать телефон незагружаемым. Никогда не прошивайте
`preloader`, `lk`, `tee`, `seccfg` и не трогайте разделы калибровок (`nvram`,
`nvdata`, `nvcfg`, `proinfo`, `protect1/2`). Сделайте полный бэкап и до любых
опытов прочитайте, как вернуться в Android ([docs/INSTALL-LINUX.ru.md](docs/INSTALL-LINUX.ru.md),
разделы 4 и 13, и [docs/DUALBOOT-ROLLBACK.md](docs/DUALBOOT-ROLLBACK.md)).

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

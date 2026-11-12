<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Сборка mainline Linux для Unihertz Titan 2

[English version](BUILD-LINUX.md) · Установка: [INSTALL-LINUX.ru.md](INSTALL-LINUX.ru.md) ·
Что работает: [STATUS.ru.md](STATUS.ru.md)

Здесь из исходников на ПК собирается всё, что нужно для установки. Телефон ни на одном шаге не
нужен. Команды выполняются в корне клона `titan2-tools`, если не сказано иное; параметры каждого
скрипта описаны в его заголовке (`sed -n 1,40p <скрипт>`).

## 1. Из чего состоит

| репозиторий | что в нём |
|---|---|
| [titan2-linux](https://github.com/Serlk/titan2-linux), ветка `titan2` | ядро (на базе mainline 7.2) и дерево устройств телефона |
| [titan2-tools](https://github.com/Serlk/titan2-tools) | скрипты сборки, initramfs, сборщик корневой ФС Alpine, службы и программы userspace (файловый и RPC-серверы модема, демон звука в звонке, плагин ModemManager, vendor-окружение для дополнительных частей) |
| [titan2-boot](https://github.com/Serlk/titan2-boot) | меню загрузки, которое запускается из `boot_a` после стокового загрузчика (LK) |
| [titan2-twrp](https://github.com/Serlk/titan2-twrp) | TWRP для Titan 2 (нужен один раз, для переразметки) |
| [titan2-camera](https://github.com/Serlk/titan2-camera) | по желанию: драйверы камер (серия патчей ядра) и патчи libcamera |
| [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | по желанию: отпечаток (серия патчей ядра и userspace); пока не работает |
| [titan2-gnss](https://github.com/Serlk/titan2-gnss) | по желанию: GNSS с A-GPS (серия патчей ядра и userspace) |
| [titan2-android](https://github.com/Serlk/titan2-android) | crDroid 16 для Titan 2 — Android-сторона двойной загрузки |
| [titan2-docs](https://github.com/Serlk/titan2-docs) | эта документация |

Что получится:

| результат | чем | куда |
|---|---|---|
| `out/mainline-build/` (ядро `Image.gz`, DTB, модули) | `scripts/build-mainline.sh` | для двух следующих |
| `out/bootmenu-dualboot/boot.img` | `scripts/make-bootmenu.sh` | раздел `boot_a` |
| `out/linux-root/linux.img` | `scripts/make-linux-root.sh` | раздел `linux` (создаётся при установке) |
| `out/t2-repo/` (по желанию) | `scripts/t2-repo.sh` (наши aports), `scripts/build-libcamera.sh` из titan2-camera | пакеты для корневой ФС |

## 2. Что нужно на ПК

- ПК с Linux (x86_64) и **Docker** (ядро, утилиты образов и пакеты собираются в контейнерах).
  Пакеты (Mesa, libcamera) собираются под aarch64 в эмуляции, для этого нужно ядро хоста 6.7 или
  новее.
- `git`, `ssh`, `adb` и `fastboot` (Android platform-tools), `img2simg` (android-tools, делает
  разреженный `linux.img`), `e2fsprogs` (`e2fsck` для образа корня, `debugfs` для заводских
  файлов), а для извлечения заводских файлов — `lpunpack` (android-tools) и `erofs-utils`
  (`fsck.erofs`).
- Место на диске: около **30 ГБ** на ядро (тулчейн 3,5 ГБ, исходники, каталог сборки около
  8 ГБ) и образ корня (около 4 ГБ); около **60 ГБ** вместе со сборкой Mesa и libcamera.
- Время: ядро — 20–60 минут (по умолчанию `JOBS=4`); Mesa в эмуляции — часы.

## 3. Исходники

```sh
git clone https://github.com/Serlk/titan2-tools
cd titan2-tools
git clone -b titan2 https://github.com/Serlk/titan2-linux linux       # дерево ядра (SRC=linux)
git clone https://github.com/Serlk/titan2-boot titan2-boot           # меню загрузки
# mkbootimg из AOSP (Apache-2.0, в titan2-tools не входит)
git clone https://android.googlesource.com/platform/system/tools/mkbootimg tools/mkbootimg
```

Если ядро только собирать, достаточно `--depth 1`. Берите `titan2-linux`, `titan2-boot` и
`titan2-tools` одной даты: меню передаёт ядру сведения о памяти и прошивках, и старое меню с
новым ядром может оставить Wi-Fi без его области памяти. **Ядрам начиная с 2026-11-12 нужен
titan2-boot от 2026-11-12 или новее** (коммит «take the reserved-memory rules from the mainline
DTB»): их DT Wi-Fi просит меню сделать из пула Wi-Fi от LK restricted DMA pool
(`titan2,lk-refs`), старые версии меню этого не делают, и драйвер Wi-Fi отказывается от пула.
Та же версия меню прокручивает список «Linux partition» и показывает до 24 пунктов (старые — 12).

Дополнительные части (раздел 5) — отдельные репозитории; нужные клонируйте рядом с
`titan2-tools` (пути ниже считают, что это `../titan2-camera`, `../titan2-fingerprint`,
`../titan2-gnss`):

```sh
cd ..
git clone https://github.com/Serlk/titan2-camera
git clone https://github.com/Serlk/titan2-fingerprint
git clone https://github.com/Serlk/titan2-gnss
cd titan2-tools
```

## 4. Тулчейн и сборочный контейнер

Ядро не собирается clang 17; нужен готовый clang AOSP **r574158** (clang 21) в
`toolchains/clang-r574158`. Это каталог `clang-r574158` в
<https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86>; gitiles отдаёт один
каталог архивом (ссылка «tgz» на странице каталога, адрес вида
`.../linux-x86/+archive/refs/heads/<ветка>/clang-r574158.tar.gz`):

```sh
mkdir -p toolchains/clang-r574158
tar -xzf clang-r574158.tar.gz -C toolchains/clang-r574158
toolchains/clang-r574158/bin/clang --version      # clang version 21...
docker build -t titan2-kbuild docker/             # контейнер, которым пользуются все скрипты
make -C tools CC=$PWD/toolchains/clang-r574158/bin/clang     # мелкие статические утилиты
```

## 5. Ядро

```sh
scripts/build-mainline.sh                  # -> out/mainline-build
```

Скрипт делает `make defconfig`, добавляет `configs/mainline-titan2.config` (и `FRAGMENTS`, см.
ниже) и собирает `Image.gz`, все `mt6878-unihertz-titan2*.dtb` дерева и модули (ставятся без
отладочных символов в `out/mainline-build/modules/lib/modules/<версия>`). Переменные:
`SRC=linux` (дерево ядра, внутри titan2-tools), `O=out/mainline-build` (каталог сборки),
`JOBS=4`, `MODULES=0` (только ядро и DTB), `FRAGMENTS="<файл> ..."` (дополнительные фрагменты
конфига, подмешиваются после базового в указанном порядке; пути относительно текущего каталога,
могут быть и вне titan2-tools).

Варианты дерева устройств (по пункту меню на каждый):

| DTB | что это |
|---|---|
| `mt6878-unihertz-titan2-all.dtb` (`all`) | **всё, что работает**: экран (mediatek-drm), GPU, Wi-Fi, Bluetooth, модем, звук, зарядник, датчики. Пользуйтесь им. |
| `mt6878-unihertz-titan2-all-fb.dtb` (`all-fb`) | то же, но экран остаётся на фреймбуфере загрузчика (simplefb). Запасной, если экран чёрный. |
| `mt6878-unihertz-titan2.dtb` (`full`) | база: экран через фреймбуфер, клавиатура, тач, кнопки, накопитель, USB, PMIC, датчик заряда. Без радио и GPU. Самый безопасный пункт. |
| `-all-iommu` | для разработки, только с серией камеры: экран за IOMMU. Показывает мусор, не использовать. |
| `-drm`, `-gpu`, `-wifi`, `-modem`, `-audio`, `-pd`, `-dvfs`, `-idle`, `-jpeg`, `-charger-*` | проверочные варианты по одной подсистеме, для разработки |

Не правьте дерево ядра во время сборки (контейнер собирает то, что лежит на диске в этот
момент), а параллельным сборкам давайте свои каталоги `O=`.

### С дополнительными частями

Камеры, отпечатка и GNSS в titan2-linux нет. Каждая часть живёт в своём репозитории, где её
ядерная часть — серия патчей поверх ветки `titan2` (`kernel/patches/*.patch`, сделана
`git format-patch` на коммите `titan2` `50a22e5003dd`) и фрагмент конфига (`kernel/<имя>.config`);
подробности — в её `kernel/README.md`. Наложите серии на локальную ветку вашего `titan2` и
передайте фрагменты сборке:

```sh
cd linux
git checkout -b titan2-local titan2           # или базовый коммит: git checkout -b titan2-local 50a22e5003dd
git am ../../titan2-camera/kernel/patches/*.patch
git am ../../titan2-fingerprint/kernel/patches/*.patch
git am ../../titan2-gnss/kernel/patches/*.patch
cd ..
FRAGMENTS="../titan2-camera/kernel/titan2-camera.config \
    ../titan2-fingerprint/kernel/titan2-fingerprint.config \
    ../titan2-gnss/kernel/titan2-gnss.config" \
    scripts/build-mainline.sh
```

Оставьте строки только нужных частей; три серии трогают разные места и накладываются одна за
другой (проверено в порядке камера, отпечаток, GNSS). Фрагмент без своей серии безвреден, но ничего
не меняет (у его опций нет записи в Kconfig, и они отбрасываются). Если `git am` останавливается на
более новом `titan2`, сделайте `git am --abort` и начните локальную ветку с указанного выше
базового коммита. Серия камеры добавляет камеры в DT `all`; серии отпечатка и GNSS добавляют свои
узлы в базовую плату и в DT Wi-Fi.

## 6. initramfs

Отдельно собирать не нужно: `make-bootmenu.sh`, `make-linux-root.sh` и `linux-update.sh` сами
вызывают `tools/initramfs/build.sh`. Он скачивает пакеты Alpine v3.24 aarch64 (busybox, dropbear,
e2fsck, …) и проверяет их подписи ключом Alpine из `tools/initramfs/keys`. Что он делает на
телефоне: сеть по USB (телефон — 172.16.42.1, ПК получает адрес по DHCP) и последовательная
консоль, SSH, поиск и проверка корня (`t2.root=part:linux`), растяжение ФС раздела `linux` при
первой загрузке и перевод всех остальных разделов UFS в режим только чтения с односторонней
блокировкой в ядре (см. [DUALBOOT.md](DUALBOOT.md), раздел «Protection»).

**Ключи SSH:** скрипты берут ваш открытый ключ из `AUTHORIZED_KEYS=<файл>`, иначе из
`tools/initramfs/authorized_keys`, иначе `~/.ssh/id_ed25519.pub`. Без ключа у root **пустой
пароль** (SSH тогда доступен по кабелю USB, а после настройки Wi-Fi — и по Wi-Fi). Всегда
передавайте ключ.

## 7. Пакеты: свой репозиторий apk (по желанию)

Пакеты, которых нет в Alpine v3.24, собираются в ваш собственный подписанный репозиторий apk
`out/t2-repo`; сборка образа корня берёт его как `LOCAL_REPO`:

```sh
scripts/t2-repo.sh keygen                  # один раз: out/t2-repo-keys/t2-repo.rsa(.pub); сохраните
make -C ../titan2-fingerprint dist; make -C ../titan2-gnss dist     # их архивы исходников
scripts/t2-repo.sh aport tools/aports/titan2-vendor-runtime \
    ../titan2-fingerprint/aports/titan2-fingerprint \
    ../titan2-gnss/aports/titan2-gnss ../titan2-gnss/aports/titan2-epo   # -> out/t2-packages
scripts/t2-repo.sh build                   # общий подписанный индекс -> out/t2-repo
scripts/t2-repo.sh list
```

`aport` собирает пакеты aarch64 в контейнере `alpine` arm64, поэтому на ПК нужна поддержка
qemu-user binfmt для aarch64 (например, пакет `qemu-user-static` вашего дистрибутива). Образ
доверяет открытому ключу `out/t2-repo/t2-repo.rsa.pub`; сохраните пару ключей, если захотите
потом обновлять пакеты на телефоне.

- **Камера**: `scripts/build-libcamera.sh build` из titan2-camera собирает libcamera с патчами
  (часы в эмуляции, ядро хоста 6.7 или новее). Подпишите тем же ключом и положите результат внутрь
  titan2-tools, чтобы `t2-repo.sh build` его проиндексировал:

  ```sh
  (cd ../titan2-camera && TITAN2_TOOLS=../titan2-tools KEYS=../titan2-tools/out/t2-repo-keys \
      OUT=../titan2-tools/out/libcamera scripts/build-libcamera.sh build)
  EXTRA_APK_DIRS=out/libcamera/packages/aports/aarch64 scripts/t2-repo.sh build
  ```

- **Отпечатку и GNSS** нужен ещё `bionic_translation` с патчами из
  `tools/titan2-vendor-runtime/patches/bionic_translation` (см. тамошний README): соберите его из
  aport Alpine с этими патчами и добавьте файлы `.apk` через `EXTRA_APK_DIRS=`, как выше.
- **Mesa**: в Alpine v3.24 нет Mesa с поддержкой Mali-G615 телефона (Valhall v11). Mesa 26.3.0 —
  первый выпуск с G615; скрипт его сборки пока не опубликован. Без него рабочий стол рисуется
  процессором (llvmpipe). Если вы собрали пакеты Mesa 26.3 сами, добавьте их через
  `EXTRA_APK_DIRS=`; сборка образа обновит пакеты Mesa из `LOCAL_REPO`.

## 8. Корневая ФС (`linux.img`)

```sh
LOCAL_REPO=out/t2-repo AUTHORIZED_KEYS=~/.ssh/id_ed25519.pub \
EXTRA_PKGS="modemmanager modemmanager-openrc networkmanager-wwan polkit polkit-openrc \
    alsa-lib alsa-utils pipewire wireplumber pipewire-pulse" \
    scripts/make-linux-root.sh
```

Результат: `out/linux-root/linux.img` (разреженный, для `fastboot flash linux`) и `linux.raw`
(обычный ext4). Скрипт собирает систему Alpine v3.24 aarch64 (внутри — `scripts/netroot-serve.sh
build`: OpenRC, eudev, NetworkManager с Wi-Fi, dropbear, chrony, службы и настройки Titan 2 из
`tools/netroot/`), ставит модули ядра из `out/mainline-build` и добавляет `/boot` (ядро,
initramfs, все DTB и `/boot/extlinux/extlinux.conf` с пунктом на каждый вариант DT, корень
`t2.root=part:linux`). Скрипты пакетов, которые нельзя выполнить на x86, выполнит первая загрузка
на телефоне.

Переменные:

| переменная | смысл |
|---|---|
| `KO=out/mainline-build` | какую сборку ядра ставить |
| `SIZE=8G` | размер ext4 до растяжения на весь раздел на телефоне |
| `VARIANTS="all all-fb full"` | для каких вариантов DT делать пункты, в порядке меню (по умолчанию `all`, `all-fb`, `full`, затем остальные собранные) |
| `DEFAULT=all` | пункт по умолчанию в `/boot/extlinux/extlinux.conf` (метки — имена вариантов) |
| `GFX=1` (по умолчанию) | графика: niri, foot, alacritty, rofi, мост Xwayland, weston/cage для проверок, iio-sensor-proxy, BlueZ |
| `LOCAL_REPO=out/t2-repo` | ваш репозиторий пакетов; его Mesa (а с `CAMERA=1` и libcamera) заменяют пакеты Alpine |
| `CAMERA=1 CAMERA_DIR=../titan2-camera` | камера: libcamera, PipeWire + WirePlumber, порталы, GNOME Snapshot, qcam (нужны наша libcamera в `LOCAL_REPO` и ядро с серией камеры) |
| `FP=1 FP_DIR=../titan2-fingerprint` | пакеты отпечатка (нужны они, `titan2-vendor-runtime` и `bionic_translation` в `LOCAL_REPO`) |
| `GNSS=1 GNSS_DIR=../titan2-gnss` | пакеты GNSS, GeoClue (требования как у `FP=1`) |
| `EXTRA_PKGS="..."` | дополнительные пакеты Alpine; в примере выше — то, что нужно модему, звонкам и звуку |
| `AUTHORIZED_KEYS=<файл>` | ключи SSH для root (см. раздел 6) |
| `REUSE=1` | взять уже собранный образ Alpine, пересобрать только `/boot` |
| `OUT=out/linux-root` | каталог результата |

`CAMERA`, `FP` и `GNSS` передают сборщику образа соответствующий репозиторий: ставятся пакеты из
его `netroot/packages`, его `netroot/install.d/*.sh` настраивают службы. Без `*_DIR` сборщик ищет
`external/titan2-<имя>` внутри titan2-tools. Все три части вместе:

```sh
LOCAL_REPO=out/t2-repo AUTHORIZED_KEYS=~/.ssh/id_ed25519.pub \
CAMERA=1 CAMERA_DIR=../titan2-camera FP=1 FP_DIR=../titan2-fingerprint GNSS=1 GNSS_DIR=../titan2-gnss \
    scripts/make-linux-root.sh
```

Что нужно каждой части потом на телефоне (файлы с вашего телефона, службы): см.
[INSTALL-LINUX.ru.md](INSTALL-LINUX.ru.md#12-по-желанию-камера-gnss-отпечаток).

## 9. Меню загрузки (`boot_a`)

В образ меню входит **ядро установленного у вас Android**, поэтому берите его со своего телефона
(бэкап `boot_a`, снятый перед установкой, см. INSTALL-LINUX.ru.md, шаг 2). Файл должен лежать
внутри каталога titan2-tools.

```sh
ANDROID_BOOT=backup/<дата>-pre-linux/boot_a.img OUT=out/bootmenu-dualboot \
    UFS=1 LINUX_ROOT=part:linux VARIANTS=full DEFAULT=android \
    scripts/make-bootmenu.sh
```

Результат: `out/bootmenu-dualboot/boot.img`. Пункты меню: Android, `Linux <версия> (boot_a)`
(запасное ядро внутри `boot_a` с корнем на разделе `linux`), «Linux partition» (пункты
`/boot/extlinux/extlinux.conf` раздела `linux`, их читает собственная читалка UFS меню),
Fastboot, Recovery, Reboot.

| переменная | смысл |
|---|---|
| `ANDROID_BOOT=<boot.img>` | обязательно: boot-образ Android, чьё ядро кладётся в меню. Никогда не `boot_a`, в котором уже стоит меню. |
| `UFS=1` | добавляет «Linux partition»; UFS читается только когда этот пункт выбран (или он по умолчанию) |
| `LINUX_ROOT=part:linux` | корень запасных пунктов Linux |
| `VARIANTS=full` | варианты DT запасных пунктов (`full` — безопасный) |
| `DEFAULT=android\|linux\|linux-<вариант>\|ufs`, `TIMEOUT=5` | пункт по умолчанию и отсчёт в секундах |
| `NO_LINUX=1` | меню только с Android |
| `CMDLINE` | командная строка ядра запасных пунктов |
| `OUT=out/bootmenu` | каталог результата |

Recovery (TWRP), зарядка выключенного телефона и загрузка по будильнику меню не показывают и
всегда запускают ядро Android, поэтому от Linux-части не зависят.

## 10. Программы для телефона, которые ещё не упакованы

Стек модема и звонков — небольшие программы на C в titan2-tools. apk-пакетов для них пока нет;
проще всего собрать их на самом телефоне (в Alpine есть компиляторы):

| программа | каталог | роль |
|---|---|---|
| `t2-ccci-fsd` | `tools/ccci-fsd` | отвечает на файловые запросы модема к NVRAM из копии-песочницы (никогда не из раздела флеш-памяти) |
| `t2-ccci-rpcd` | `tools/ccci-rpcd` | отвечает на RPC-запросы модема (образ модема читает только на чтение) |
| `t2-speechd` | `tools/speechd` | звук в звонке: аудиотракт AP и речевой движок модема |
| плагин ModemManager `mtk-ccci` | `tools/mm-titan2` | звонки, SMS, мобильный интернет, IMS (VoLTE) через ModemManager |
| `t2-powerd` (по желанию) | `tools/powerd` | экспериментальный менеджер сна |

В каждом каталоге README со сборкой, тестами и запуском; `make test` запускает тесты на ПК.
Установка стека модема на телефон — [INSTALL-LINUX.ru.md, раздел 10](INSTALL-LINUX.ru.md#10-мобильная-сеть-звонки-и-sms).

## 11. Проверки без телефона

```sh
scripts/test-gpt-carve.sh           # переразметка на образе с разметкой GPT Titan 2
scripts/test-bootmenu-qemu.sh       # меню загрузки в qemu (Android, Linux, recovery, пункты с диска)
tools/initramfs/qemu/diskroot.sh    # корень на разделе, блокировка read-only, попытки записи от root
```

Собственные тесты меню описаны в README titan2-boot (`make test`, `make qemu`).

Дальше: [INSTALL-LINUX.ru.md](INSTALL-LINUX.ru.md).

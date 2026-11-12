<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Установка mainline Linux на Unihertz Titan 2 (две системы с Android)

[English version](INSTALL-LINUX.md) · Сборка: [BUILD-LINUX.ru.md](BUILD-LINUX.ru.md) ·
Что работает: [STATUS.ru.md](STATUS.ru.md) · Подробности двойной загрузки:
[DUALBOOT.md](DUALBOOT.md) · Как всё вернуть: [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md)
(оба на английском)

## 1. Что получится

- Меню загрузки после собственного загрузчика телефона: **Android** или **Linux**, выбор
  клавишами громкости. Android, его recovery и зарядка выключенного телефона работают как прежде.
- Mainline **Linux 7.2** с системой **Alpine Linux** (v3.24, aarch64) на собственном разделе
  (`linux`, по умолчанию 256 ГиБ, вырезается из конца `userdata` Android).
- В Linux: экран, клавиатура, тачскрин и тачпад, Wi-Fi, мобильный интернет, звонки VoLTE и SMS,
  Bluetooth (базово), GPU (OpenGL ES и Vulkan с нашей сборкой Mesa), зарядка, датчики, а по
  желанию — камеры и GNSS. Подробности и то, что не работает, — в [STATUS.ru.md](STATUS.ru.md).
- Linux **не может писать в разделы Android**: всё на UFS, кроме раздела `linux`, переводится в
  режим только чтения с блокировкой, которую до следующей загрузки не снимет даже root.

Проект для тех, кому привычны `fastboot`, `adb`, SSH и командная строка. Установщика и
графической настройки нет; собственное окружение рабочего стола разрабатывается и будет
опубликовано позже. Пока в образе есть Wayland-композитор **niri** — им мы и пользуемся на
телефоне; другие Wayland-композиторы из Alpine (phosh, sxmo, sway, …) должны работать на том же
стеке DRM/GPU, но мы их не проверяли.

## 2. Риски и отказ от ответственности

- Установка **один раз стирает данные Android** (без этого не переразметить). Сначала сохраните
  из Android всё нужное.
- Ошибка при прошивке может сделать телефон незагружаемым. Всё описанное проделано на нашем
  телефоне и обратимо, пока целы бэкапы и вы не трогаете разделы из раздела 4.
- Разблокированный загрузчик и неофициальное ПО могут лишить вас гарантии и ослабляют
  защищённость телефона (любой, у кого он в руках, может его перепрошить).
- Это независимый проект сообщества, он не связан с Unihertz и MediaTek. Никаких гарантий нет;
  всё вы делаете на свой риск.

## 3. Что нужно

- **Unihertz Titan 2** (MT6878) на стоковой прошивке **V01.00.14**, с **разблокированным
  загрузчиком** (`fastboot flashing unlock`, телефон при этом стирается) и выключенной проверкой в
  `vbmeta_a`. Через оба шага проводит установка [crDroid для Titan 2](https://github.com/Serlk/titan2-android);
  двойная загрузка проверена с crDroid 16 в роли Android. Стоковый Android должен работать так же
  (меню берёт ядро того Android, который у вас стоит), но мы этого не пробовали.
- Активен **слот A**: `fastboot getvar current-slot` должен ответить `a`.
- **TWRP для Titan 2** в `vendor_boot_a`: [titan2-twrp](https://github.com/Serlk/titan2-twrp), см.
  его `docs/INSTALL.ru.md`. Он нужен для бэкапов и переразметки.
- ПК с Linux, Docker, git, `adb`/`fastboot` (свежие platform-tools), `img2simg`, `ssh`,
  `e2fsprogs`, около 30 ГБ свободного места (60 ГБ со сборкой Mesa) и образы из
  [BUILD-LINUX.ru.md](BUILD-LINUX.ru.md).
- Телефон на 512 ГБ: при размере `linux` по умолчанию (256 ГиБ) Android остаётся около 210 ГиБ.
  Размер задаётся параметром (`SIZE=`, Android должно остаться не меньше 64 ГиБ).
- Хороший USB-кабель, заряд батареи больше 50 %.

## 4. Чего не делать никогда

- Не прошивать и не стирать `preloader`, `lk`, `tee`, `scp`, `seccfg`, `nvram`, `nvdata`, `nvcfg`,
  `protect1`, `protect2`, `proinfo`, `md1img`: в них загрузчики, ключи и калибровки (без них нет
  связи и IMEI).
- Не переключаться на слот B (`fastboot set_active b`, `--slot b`): там старые загрузчики.
- **Не ставить OTA-обновления Android**: они пишут слот B и переключаются на него.
- Не блокировать загрузчик, пока стоит что-то кроме подписанной стоковой прошивки.
- Не выполнять `fastboot -w` в fastbootd (где `fastboot getvar is-userspace` отвечает `yes`),
  кроме как ровно в порядке шага 5.

**Как попасть в fastboot** в любой момент: в меню загрузки выбрать **Fastboot**; из Android —
`adb reboot bootloader`; из Linux — `t2-reboot bootloader`. Если ничего не грузится: держать
**питание около 15 с**, пока телефон не выключится, затем зажать **питание + Func1**
до меню загрузчика («Select Boot Mode») и выбрать **Fastboot Mode** (громкость вверх —
перемещение, громкость вниз — выбор). `fastboot devices` должен показать телефон.

## 5. Установка

Выполняйте по одному шагу и проверяйте результат перед следующим. Команды — на ПК, в каталоге
titan2-tools; `<дата>` — любое имя каталога бэкапа на ваш выбор.

### Шаг 1. Android и TWRP

Поставьте Android (crDroid или стоковый V01.00.14) и TWRP по их собственным инструкциям.
Загрузите Android и убедитесь, что он работает, затем загрузите TWRP (Recovery в меню
загрузчика или `adb reboot recovery`). **Не расшифровывайте `/data` в TWRP** на следующих шагах
(запрос пароля — отменить).

### Шаг 2. Бэкапы

В TWRP, когда работает `adb`:

```sh
mkdir -p backup/<дата>-pre-linux && cd backup/<дата>-pre-linux
for p in boot_a init_boot_a vendor_boot_a dtbo_a vbmeta_a misc \
         protect1 protect2 nvdata nvcfg nvram proinfo seccfg; do
    adb pull /dev/block/by-name/$p $p.img
done
sha256sum *.img > SHA256SUMS
cd ../..
```

Храните этот каталог бережно, в том числе не только на этом ПК. `boot_a.img` — ядро вашего
Android для меню загрузки; `nvdata`, `nvcfg`, `protect1`, `protect2` — ещё и источник копии NVRAM
модема в Linux (раздел 9). Бэкап личный: в нём ваш IMEI и ключи, никому его не передавайте.

### Шаг 3. Собрать образы

По [BUILD-LINUX.ru.md](BUILD-LINUX.ru.md): ядро, `out/linux-root/linux.img` и меню загрузки,
собранное с `ANDROID_BOOT=backup/<дата>-pre-linux/boot_a.img` (`out/bootmenu-dualboot/boot.img`).

### Шаг 4. Только меню загрузки

```sh
adb reboot bootloader
fastboot getvar current-slot          # a
fastboot flash boot_a out/bootmenu-dualboot/boot.img
fastboot reboot
```

Проверьте, прежде чем идти дальше:

1. После логотипа загрузчика появляется меню и отсчитывает до Android; Android загружается.
2. Recovery из меню запускает TWRP; зарядка выключенного телефона работает.
3. «Linux partition» показывает `UFS: LU2 ... GiB, GPT ok` и ошибку `no partition "linux"`
   (раздела ещё нет); после этого Android из того же меню по-прежнему загружается.

Если что-то не так: `fastboot flash boot_a backup/<дата>-pre-linux/boot_a.img` возвращает
состояние до этого шага (больше ничего ещё не менялось).

### Шаг 5. Переразметка и стирание

Загрузите TWRP, ничего не расшифровывайте, затем на ПК:

```sh
scripts/dualboot-repartition.sh check     # читает и проверяет GPT, ничего не пишет
scripts/dualboot-repartition.sh run       # сохраняет GPT, вырезает «linux», пишет после «yes»
```

`run` сохраняет начало и конец диска в `backup/<дата>-pre-repartition/` (нужно для отката),
пишет резервную GPT, затем основную и читает обе обратно. `SIZE=128GiB` перед командой задаёт
другой размер (по умолчанию 256GiB). Меняется только таблица разделов; `userdata` становится
короче, а данные Android стираются следующим шагом.

Стереть данные Android **именно в таком порядке** (при другом Android со второй загрузки уходит
в бесконечную перезагрузку):

```sh
adb reboot bootloader
fastboot reboot fastboot
fastboot format:f2fs metadata
fastboot reboot bootloader
fastboot -w
```

### Шаг 6. Прошить Linux

```sh
fastboot flash linux out/linux-root/linux.img
fastboot reboot
```

Размер нового раздела загрузчик не сообщает (`getvar partition-size:linux` выдаёт ошибку), но
`fastboot flash linux` его находит. Android теперь начинается с чистого листа: настройте его
заново (менеджеры root и приложения ставятся снова).

## 6. Первая загрузка Linux

1. Перезагрузитесь, в меню выберите **Linux partition**, затем пункт **Linux … -all** (в свежем
   образе по умолчанию стоит базовый пункт `Linux <версия>`, т. е. `full`, в нём нет Wi-Fi; в
   `all` есть всё).
2. Первая загрузка выполняет скрипты пакетов и в фоне растягивает ФС на весь раздел; это
   занимает несколько минут. На экране — вывод текстовой консоли.
3. Подключите USB-кабель. Телефон появится как сетевой USB-адаптер и выдаст ПК адрес
   (172.16.42.2) по DHCP:
   ```sh
   ssh root@172.16.42.1
   ```
4. На телефоне:
   ```sh
   passwd                                   # пароль root для консоли
   nmcli device wifi list
   nmcli device wifi connect "<SSID>" password "<пароль>"
   date                                     # chrony выставит время, когда появится сеть
   ```
   При следующих загрузках Wi-Fi подключается сам; тогда телефон доступен по SSH и по своему
   адресу в Wi-Fi.

По желанию — интернет для телефона через ПК по USB: телефон отправляет трафик через 172.16.42.2
с меньшим приоритетом, чем Wi-Fi. На ПК для этого нужны пересылка и NAT, например
`sysctl -w net.ipv4.ip_forward=1` и
`iptables -t nat -A POSTROUTING -s 172.16.42.0/24 -j MASQUERADE` (потом уберите их).

## 7. Меню загрузки

- **Громкость вверх/вниз** — перемещение, **питание** — запуск пункта (если клавиша питания не
  читается, запускает громкость вниз). До пункта по умолчанию идёт обратный отсчёт.
- Пункты: **Android**; **Linux <версия> (boot_a)** — запасное ядро, хранящееся в `boot_a`, с
  корнем на `linux`; **Linux partition** — пункты `/boot/extlinux/extlinux.conf` раздела `linux`
  (`all`, `all-fb`, `full`, проверочные варианты, `previous`); **Fastboot**; **Recovery**;
  **Reboot**.
- При загрузке в recovery, зарядке выключенного телефона и по будильнику меню пропускается и
  сразу запускается Android.
- Пункт по умолчанию самого меню задаётся при сборке (`DEFAULT=android|linux|ufs`, `TIMEOUT=`;
  `DEFAULT=ufs` сразу грузит пункт по умолчанию с раздела). Пункт по умолчанию внутри «Linux
  partition» — строка `DEFAULT <метка>` в `/boot/extlinux/extlinux.conf`; поправьте её на
  телефоне или дайте её выставить `linux-update.sh` (`DEFAULT=all`).
- titan2-boot начиная с 2026-11-12 показывает до 24 пунктов раздела и прокручивает список;
  старые версии меню показывают только первые 12. `linux-update.sh` и `make-linux-root.sh`
  ставят `all`, `all-fb`, `full` первыми; `VARIANTS="..."` сокращает список.
- Если читалка UFS меню зависла, держите питание 15 с; Android работает (он читалкой не
  пользуется), а её журнал после сброса — в `/sys/fs/pstore/console-ramoops-0` в Android.
- Новое ядро Android (обновление Android, меняющее `boot`): пересоберите меню с
  `ANDROID_BOOT=<новый boot.img>` и прошейте `boot_a`. `boot.img` Android, прошитый поверх
  `boot_a`, убирает меню (Android при этом грузится; меню возвращается его повторной прошивкой).

## 8. Обновление ядра

Соберите новое ядро на ПК (`scripts/build-mainline.sh`), загрузите телефон в Linux с раздела,
подключите по USB (или Wi-Fi), затем:

```sh
scripts/linux-update.sh                       # по умолчанию HOST=root@172.16.42.1
HOST=root@<адрес-в-wifi> scripts/linux-update.sh
REBOOT=1 scripts/linux-update.sh              # и затем перезагрузить
```

Скрипт копирует на телефон ядро, все DTB, свежий initramfs и модули и переписывает
`/boot/extlinux/extlinux.conf` (пункты `all`, `all-fb`, `full`, остальные варианты и
`previous`). Прежние ядро, DTB и initramfs переезжают в `/boot/prev` (пункт **previous**), а
старые каталоги модулей остаются, так что от незагружающегося ядра до прежнего — один пункт
меню. Пункты других ядер, положенных вами в `/boot/<имя>/`, сохраняются. Ничего не прошивается;
`boot_a` и Android не трогаются.

Параметры: `KO=<каталог сборки>`, `VARIANTS="all all-fb full ..."`, `DEFAULT=all`, `CMDLINE=...`,
`SSH_OPTS="-i <ключ>"`, `FIRMWARE=1` (раздел 9). Параметры ядра, которые должны пережить
обновления, кладутся в `/boot/extlinux/cmdline.extra` на телефоне (одной строкой); скрипт
дописывает их в каждый пункт. Запускайте скрипт без фильтров: через `head` он обрывается на
середине.

Запасное ядро внутри `boot_a` обновляется только пересборкой и прошивкой меню; за каждым
обновлением ему следовать не нужно (оно работает со своими модулями на разделе).

## 9. Заводские файлы с вашего телефона

Некоторым частям нужны файлы, которые принадлежат вашему телефону: прошивки и настройки
производителя и калибровки конкретного аппарата. **Проект их никогда не распространяет, и вы
свои тоже не передавайте.** Вы копируете их со своего телефона (или из официального пакета
прошивки своей версии) для того же самого телефона.

| что | для чего | откуда |
|---|---|---|
| `frsm-spk1.bin` | усилитель громкого динамика | `vendor/firmware/` раздела vendor |
| `WIFI` (MAC-адрес и радиокалибровка Wi-Fi) | Wi-Fi с вашим MAC и калибровкой | `/APCFG/APRDEB/WIFI` в `nvdata` |
| `BT_Addr` | ваш адрес Bluetooth | `/APCFG/APRDEB/BT_Addr` в `nvdata` |
| копия NVRAM модема | модем (IMEI, калибровки) | бэкапы `nvdata`, `nvcfg`, `protect1`, `protect2` из шага 2 |
| файлы vendor, system, system_ext | GNSS, отпечаток, камера | логические разделы `vendor_a`, `system_a`, `system_ext_a` из `super` |

Прошивки радиочасти Wi-Fi/BT, модема и концентратора датчиков загрузчик телефона берёт из его
собственных разделов, ровно как для Android; копии для Linux не нужны.

**Разделы vendor.** В TWRP (или в Android с root). `adb shell ls /dev/block/mapper` должен
показать `vendor_a`; если нет — один раз смонтируйте Vendor на экране Mount в TWRP (тогда TWRP
его подключит).

```sh
mkdir -p out/stock-super && cd out/stock-super
adb pull /dev/block/mapper/vendor_a vendor_a.img      # при нужде также system_a, system_ext_a
# вместо TWRP — Android с root:
#   adb exec-out su -c 'cat /dev/block/mapper/vendor_a' > vendor_a.img
fsck.erofs --extract=vendor vendor_a.img            # erofs-utils; образ ext4 — через debugfs rdump
cd ../..
```

Или возьмите раздел `super` целиком (`adb pull /dev/block/by-name/super`, несколько ГБ, или
`super.img` из официального пакета прошивки вашей версии) и разберите его `lpunpack super.img`.
Учтите: если у вас crDroid, `system_a` — от crDroid; частям, которым нужны стоковые файлы system
(отпечаток, GNSS), они нужны из стокового пакета или из снимка, сделанного раньше.

**Файлы звука, Wi-Fi и Bluetooth** на телефон кладёт `linux-update.sh` (`FIRMWARE=1` берёт
`frsm-spk1.bin` из `out/*-super/*/firmware/`, т. е. из распаковки выше, а `WIFI`/`BT_Addr` — из
самого свежего `backup/*/nvdata.img` или из `NVDATA_IMG=<файл>`), либо вручную:

```sh
debugfs -R "dump /APCFG/APRDEB/WIFI WIFI" backup/<дата>-pre-linux/nvdata.img
debugfs -R "dump /APCFG/APRDEB/BT_Addr BT_Addr" backup/<дата>-pre-linux/nvdata.img
ssh root@172.16.42.1 mkdir -p /lib/firmware/mediatek/mt6878
scp WIFI BT_Addr root@172.16.42.1:/lib/firmware/mediatek/mt6878/
scp out/stock-super/vendor/firmware/frsm-spk1.bin root@172.16.42.1:/lib/firmware/
```

## 10. Мобильная сеть, звонки и SMS

Модем работает на своей прошивке (её грузит загрузчик); Linux должен отвечать на его файловые
запросы из копии NVRAM телефона, запустить его и передать ModemManager. Это пока не упаковано и
требует нескольких ручных шагов. Полное описание части ModemManager — `tools/mm-titan2/INSTALL.md`
в titan2-tools (на английском).

1. **Параметры ядра.** На телефоне:
   ```sh
   echo "mtk_ccci.power_ctrl=1 mtk_ccci.runtime_data=1 mtk_ccci.wwan_ports=1" > /boot/extlinux/cmdline.extra
   ```
   затем один раз запустите `scripts/linux-update.sh` с ПК (он допишет строку в каждый пункт) и
   перезагрузитесь в `all`.
2. **Песочница NVRAM** (копия; модем никогда не пишет в раздел флеш-памяти). На ПК, из бэкапа
   шага 2 (для loop-монтирования нужен root):
   ```sh
   B=backup/<дата>-pre-linux; S=out/modem-sandbox/ro; mkdir -p $S /tmp/t2m
   for p in nvdata:nvdata protect1:protect_f protect2:protect_s nvcfg:nvcfg; do
       sudo mount -o ro,loop,noload $B/${p%%:*}.img /tmp/t2m
       mkdir -p $S/${p#*:} && sudo cp -a /tmp/t2m/. $S/${p#*:}/
       sudo umount /tmp/t2m
   done
   sudo tar -C out/modem-sandbox -czf modem-sandbox.tgz ro && sudo rm -rf out/modem-sandbox
   scp modem-sandbox.tgz root@172.16.42.1:/root/
   ```
   На телефоне:
   ```sh
   adduser -S -D -H -h /var/lib/titan2-modem t2fsd
   mkdir -p /var/lib/titan2-modem && tar -C /var/lib/titan2-modem -xzf /root/modem-sandbox.tgz
   chown -R t2fsd /var/lib/titan2-modem && rm /root/modem-sandbox.tgz
   ```
   Песочница — это ваш IMEI и калибровки: держите её на телефоне, никому не передавайте.
3. **Серверы модема.** На телефоне, с клоном titan2-tools (git или `scp -r`):
   ```sh
   apk add build-base alsa-lib-dev git
   make -C titan2-tools/tools/ccci-fsd && make -C titan2-tools/tools/ccci-rpcd
   make -C titan2-tools/tools/speechd
   install -m 755 titan2-tools/tools/ccci-fsd/t2-ccci-fsd titan2-tools/tools/ccci-rpcd/t2-ccci-rpcd \
       titan2-tools/tools/speechd/t2-speechd /usr/local/bin/
   ```
   В `/etc/conf.d/t2-modem` поставьте `T2_MODEM_RADIO=` (пусто: радио включает ModemManager),
   затем:
   ```sh
   rc-service t2-modem start                  # ждёт «modem state: ready»
   head -1 /sys/kernel/debug/d180000.mddriver/fsm
   rc-update add t2-modem default
   ```
4. **ModemManager с плагином mtk-ccci**: пакеты, сборка плагина
   (`tools/mm-titan2/build-on-phone.sh` на телефоне или кросс-сборка), правило udev и файл sysctl —
   как в `tools/mm-titan2/INSTALL.md`. Затем:
   ```sh
   rc-update add modemmanager default; rc-service modemmanager start
   rc-update add t2-speechd default;   rc-service t2-speechd start
   mmcli -m any                               # SIM-карты, регистрация, технология доступа
   ```
5. **Мобильный интернет** через NetworkManager:
   ```sh
   nmcli connection add type gsm ifname '*' con-name mobile gsm.apn <APN вашего оператора>
   nmcli connection up mobile
   ```
6. **Звонки и SMS** — `mmcli` (или любой интерфейс к ModemManager):
   ```sh
   mmcli -m any --voice-create-call="number=<номер>"   # печатает путь звонка, напр. .../Call/1
   mmcli -o 1 --start                                         # набрать; --hangup — завершить
   mmcli -m any --voice-list-calls; mmcli -o <n> --accept    # входящий
   mmcli -m any --messaging-create-sms="text='Привет',number='<номер>'"
   mmcli -s <n> --send; mmcli -m any --messaging-list-sms
   ```
   Звук в звонке переключает WirePlumber на профиль карты «Voice Call», поэтому PipeWire и
   WirePlumber должны работать в вашей сессии (сессия niri из раздела 11 их запускает).

Предпочтение сети по умолчанию — сначала 5G/LTE (меняется через
`mmcli -m any --set-allowed-modes=... --set-preferred-mode=...`). VoLTE нужен LTE; IMS
настраивает плагин.

## 11. Рабочий стол

```sh
adduser <пользователь>                               # на телефоне
for g in video input seat audio; do adduser <пользователь> $g; done
# /etc/conf.d/t2-niri: T2_NIRI_USER=<пользователь>
rc-update add t2-niri default; rc-service t2-niri start
```

`t2-niri` запускает niri с сессией D-Bus, PipeWire и WirePlumber на экране телефона. Настройки
niri — в `~/.config/niri/config.kdl`. О клавиатуре: на клавиатуре Titan 2 нет Esc; **Sym**
приходит в Wayland как `KEY_PROG3` (XKB `XF86Launch3`), **Fn** — как `KEY_PROG4`
(`XF86Launch4`), назначьте их в композиторе. `T2_GSK_RENDERER=gl` в `/etc/conf.d/t2-niri`
переводит программы GTK 4 с Vulkan на OpenGL ES, если они ведут себя странно.

## 12. По желанию: камера, GNSS, отпечаток

Каждая из этих частей живёт в своём репозитории — с серией патчей ядра, своим userspace и своим
README (как наложить серию, фрагмент конфига, пакеты, службы). Они отделены нарочно. Всем им нужны
файлы **с вашего собственного телефона** (раздел 9), и эти файлы остаются на вашем телефоне:
прошивок и настроек производителя нет ни в одном нашем репозитории и образе.

| часть | репозиторий | состояние | что нужно |
|---|---|---|---|
| Камеры (задние IMX766 и Hi-847, фронтальная S5KGD1SP) | [titan2-camera](https://github.com/Serlk/titan2-camera) | работает: снимки в GNOME Snapshot / qcam / `cam`, обработка изображения программная; записи видео пока нет | ядро `titan2` + `kernel/patches` репозитория (`git am`) и `kernel/titan2-camera.config` (BUILD-LINUX.ru.md, раздел 5); libcamera из `scripts/build-libcamera.sh` того же репозитория; образ корня с `CAMERA=1`; для фронтального сенсора — его патч прошивки, сделанный из прошивки вашего собственного устройства (не распространяется; см. `kernel/README.md` репозитория) |
| GNSS + A-GPS | [titan2-gnss](https://github.com/Serlk/titan2-gnss) | работает: фикс, координаты через GeoClue; данные A-GPS скачиваются при работе; время до первого фикса ещё не замерено | ядро `titan2` + `kernel/patches` репозитория и `kernel/titan2-gnss.config`; vendor-окружение из titan2-tools; демон GNSS и HAL из ваших `vendor_a`/`system_a` |
| Отпечаток | [titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint) | **пока не работает**: TEE и HAL запускаются, но HAL не реагирует на прерывание касания, палец не определяется | ядро `titan2` + `kernel/patches` репозитория и `kernel/titan2-fingerprint.config`; vendor-окружение; клиент TEE и HAL из ваших стоковых `vendor_a`/`system_a` |

Vendor-окружение (`t2-vendor-setup` из titan2-tools) собирает закрытый каталог
(`/var/lib/titan2-vendor`) из распакованных разделов раздела 9, например
`t2-vendor-setup /root/stock`, где в `/root/stock` лежат `system/`, `system_ext/` и `vendor/`.
Ничего не скачивается, и результат нельзя копировать на чужой телефон.

## 13. Как вернуть

- **Только меню** (Android не тронут): `fastboot flash boot_a backup/<дата>-pre-linux/boot_a.img`.
- **Всю двойную загрузку** (прежняя таблица разделов, Android стирается ещё раз): из TWRP
  `scripts/dualboot-repartition.sh restore backup/<дата>-pre-repartition`, затем стирание в
  порядке шага 5, затем меню, как выше. Полностью: [DUALBOOT-ROLLBACK.md](DUALBOOT-ROLLBACK.md).
- **Полностью на сток**: после этого прошейте официальный пакет V01.00.14 утилитой
  производителя (режим Download, **не** «Format All») или прошейте из него `super`, `vbmeta*`,
  `boot`, `init_boot`, `vendor_boot`, `dtbo` через fastboot и сотрите данные. Блокировать
  загрузчик — только после этого, если вообще нужно.
- Если пропали связь, IMEI или MAC Wi-Fi (не должно: эти разделы никогда не пишутся): проверьте
  бэкап `sha256sum -c SHA256SUMS` и прошейте раздел обратно из fastboot, например
  `fastboot flash nvdata backup/<дата>-pre-linux/nvdata.img`.

## 14. Если что-то не так

**Экран остаётся чёрным после выбора Linux.** При первой загрузке подождите две-три минуты.
Попробуйте SSH по USB: если он работает, система поднялась и дело только в экране —
перезагрузитесь и выберите `all-fb` (фреймбуфер загрузчика) или `full`. Если не грузится новое
ядро — выберите `previous`. Зависший телефон перезагружается удержанием питания 15 с.

**Меню нет, сразу грузится Android.** При recovery, зарядке и будильнике меню пропускается
нарочно. Иначе в `boot_a` меню больше нет (его перезаписала прошивка или обновление Android):
прошейте `out/bootmenu-dualboot/boot.img` заново.

**«Linux partition» выдаёт ошибку.** `no partition "linux"` — переразметка не сделана (или
отменена). Ошибка UFS — выберите `Linux … (boot_a)`: он монтирует тот же корень драйвером самого
ядра. Зависание на «Starting the UFS...» — держите питание, Android это не затрагивает.

**Нет сети по USB.** Нужен кабель с данными; на ПК в `dmesg` должно появиться устройство
`cdc_ncm`. initramfs ждёт раздел-корень до 60 с и, не найдя его, остаётся доступен по USB (SSH).
На некоторых ПК новое проводное подключение надо перевести на DHCP («автоматически»).

**Нет Wi-Fi.** В пункте `full` Wi-Fi нет — загрузите `all`. Затем `nmcli radio wifi on`,
`rfkill list`, `dmesg | grep -iE 'mt7921|conninfra'`. Без файла `WIFI` (раздел 9) нет калибровки
и MAC-адреса вашего телефона.

**Модем не регистрируется.** `head -1 /sys/kernel/debug/d180000.mddriver/fsm` должен показать
`state: ready`. Если нет: в `/proc/cmdline` нет трёх параметров `mtk_ccci.*`, нет песочницы в
`/var/lib/titan2-modem/ro`, нет пользователя `t2fsd` или серверов в `/usr/local/bin` (их журналы —
в `/var/log/t2-modem/`). После неудачного запуска автомат состояний модема остаётся в `failed`
до перезагрузки. Когда модем готов: `mmcli -m any` и журнал ModemManager
(`ModemManager --debug`); `mmcli -m any -e` включает модем. Звонкам VoLTE нужны покрытие LTE и
регистрация IMS; мобильному интернету — правильный APN.

**Громкий динамик молчит.** Нет `frsm-spk1.bin` в `/lib/firmware` (раздел 9).

**Bluetooth не появляется.** Его драйвер пока не загружается сам: `modprobe btmtkbgf`, затем
`bluetoothctl`.

**TWRP загружается снова и снова.** TWRP сам снимает флаг recovery через 10–20 с после старта;
не перезагружайте его раньше. Иначе — TWRP, Reboot, System.

**Батарея садится во сне.** Известно: сон работает, но потребляет около 80 мА, см.
[STATUS.ru.md](STATUS.ru.md).

**Что-то не так внутри Linux.** Не чините Android из Linux (писать туда он всё равно не может).
Перезагрузитесь (`t2-reboot bootloader` — в fastboot) и пользуйтесь Android или TWRP.

Журналы, которые вы выкладываете публично (`dmesg`, `mmcli`), содержат серийный номер, IMEI,
идентификаторы SIM и MAC-адреса; уберите их перед публикацией.

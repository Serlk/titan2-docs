<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# What works: mainline Linux on the Unihertz Titan 2

[Русская версия](STATUS.ru.md) · Installing: [INSTALL-LINUX.md](INSTALL-LINUX.md) ·
Building: [BUILD-LINUX.md](BUILD-LINUX.md)

State of **2026-11-12**: kernel 7.2 (titan2-linux, branch `titan2`), DT variant `all`, Alpine
v3.24, tested on one phone (Titan 2, firmware V01.00.14, crDroid 16 as the Android side).
"Works" means it was used on that phone; it does not mean it was tested in every situation.

**Works** · **Partly**: works with limits, or only parts were tested · **Not yet** · **Not tested**: may work, nobody has tried

## Phone functions

| function | state | notes |
|---|---|---|
| VoLTE calls, outgoing and incoming | **Works** | through ModemManager (our `mtk-ccci` plugin), AMR-WB, audio both ways through the earpiece and the microphone; IMS set up by the plugin; both SIMs register. Not tested: emergency calls, calls after every kind of cold boot, incoming on SIM 2, loudspeaker in a call |
| Calls without LTE (2G/3G, CSFB) | **Not tested** | |
| SMS | **Works** | receiving and sending with the screen on. Receiving while the phone sleeps: not verified. SMS on SIM 2 are not read yet |
| MMS | **Not yet** | |
| Mobile data | **Works** | LTE through NetworkManager, about 17-36 Mbit/s measured; network preference 5G/LTE first (5G not verified on the air). IPv4 only; data on SIM 2 not tested; data right after waking from sleep not verified |
| Waking up for a call or SMS while asleep | **Partly** | the modem wakes the system in the kernel; the whole path (sleep, ring, screen on) has not been verified end to end |

## Connectivity

| function | state | notes |
|---|---|---|
| Wi-Fi | **Works** | 2.4 and 5 GHz, WPA2, NetworkManager, connects at boot, survives sleep; the firmware wake filter lets the phone sleep on a Wi-Fi network and wakes it for traffic to it (ping, SSH). Hotspot not tested |
| Bluetooth | **Partly** | power on, scanning. The driver is loaded by hand (`modprobe btmtkbgf`); headsets (A2DP/HFP) not verified |
| USB | **Works** | device mode: network (172.16.42.1) and serial console. USB host (OTG) is not enabled |
| NFC, infrared | **Not yet** | |

## Display, input, sound

| function | state | notes |
|---|---|---|
| Main display | **Works** | 1440x1440 command-mode DSI panel with mediatek-drm, backlight, on/off (DPMS). The display IOMMU is off (with it on, the picture is garbage); `all-fb` keeps the bootloader framebuffer as a fallback |
| GPU | **Works** | Mali-G615 with panthor; OpenGL ES with panfrost and Vulkan with panvk need our Mesa 26.3 build (the first Mesa release with this GPU); without it, software rendering |
| Keyboard, touchscreen, keyboard touchpad | **Works** | Sym and Fn reach Wayland as KEY_PROG3/KEY_PROG4; no Esc key on the hardware |
| Buttons, vibration | **Works** | power, volume, side keys; vibration motor |
| Loudspeaker, earpiece, microphone | **Works** | loudspeaker needs `frsm-spk1.bin` from your phone (INSTALL-LINUX.md section 9); earpiece and microphone are used in calls |
| Headset over USB-C | **Not yet** | |
| Rear (second) display | **Not yet** | |
| Notification LED | **Not yet** | |

## Power

| function | state | notes |
|---|---|---|
| Battery level | **Works** | SH366100 fuel gauge: percentage, current, voltage, temperature |
| Charging | **Works** | USB PD (9 V contract) and USB-A chargers at 1.5 A into the battery, PC ports with BC1.2 detection; charging also in sleep (500 mA). Charge voltage 4.35 V (stock uses 4.45 V, so slightly less capacity); battery temperature window 5-45 °C with a reduced current below 15 °C |
| Sleep (s2idle) | **Partly** | enters and resumes reliably, wakes on RTC alarm, modem and Wi-Fi; the phone draws **about 80 mA asleep** because the deepest state (26 MHz clock off) is not reached yet. Automatic sleep management is experimental |
| CPU frequency and thermal | **Works** | eight cores up to 2.5 GHz, thermal zones, CPU idle states |

## Sensors, camera, location, security

| function | state | notes |
|---|---|---|
| Accelerometer, light, proximity | **Works** | through the sensor hub and iio-sensor-proxy (`monitor-sensor`); calibration not applied (light values are uncalibrated) |
| Cameras | **Works** ([titan2-camera](https://github.com/Serlk/titan2-camera)) | all three (IMX766, Hi-847, S5KGD1SP); photos with GNOME Snapshot, qcam, `cam`; image processing in software. No video recording (no hardware video codec yet) |
| GNSS | **Works** ([titan2-gnss](https://github.com/Serlk/titan2-gnss)) | fixes with many satellites, position through GeoClue, A-GPS data download. Time to first fix not measured; the speed value was wrong in the last test |
| Fingerprint | **Not yet** ([titan2-fingerprint](https://github.com/Serlk/titan2-fingerprint)) | the TEE and the sensor HAL start, the touch interrupt arrives in the kernel, but the HAL ignores it, so no finger is detected |
| Hardware video decode/encode | **Not yet** | |

## Storage and system

| function | state | notes |
|---|---|---|
| UFS storage | **Works** | read-write on the `linux` partition only; every other partition is read-only with a lock root cannot lift |
| Dual boot with Android | **Works** | the titan2-boot menu: Android, recovery (TWRP), power-off charging, Linux from `boot_a` and from the `linux` partition |
| Kernel updates | **Works** | `linux-update.sh` from the PC, with the previous kernel kept as a menu entry |
| Desktop | **Partly** | niri runs on the GPU; our own desktop environment is being developed and will be published later; other Wayland compositors are untested |
| Android apps | **Partly** | Android Translation Layer runs some apps; experimental |

## Known issues

- Sleep current about 80 mA (see above): expect a much shorter standby time than on Android.
- The display must not be put behind the IOMMU (`all-iommu` entry): garbage on the screen.
- After a failed modem start, the modem stays in the `failed` state until a reboot.
- SMS: messages to SIM 2 are not read; delivery while asleep is not verified.
- Bluetooth driver not loaded automatically; Bluetooth and Wi-Fi together at boot are not
  verified.
- The menu shows only the first entries of `extlinux.conf` (12 or more, depending on the
  titan2-boot version).
- Logs (`dmesg`, `mmcli`) contain your serial number, IMEI and MAC addresses.

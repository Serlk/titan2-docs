<!-- SPDX-License-Identifier: GPL-2.0-or-later -->
# Touchscreen and keyboard touchpad (Synaptics RMI4)

Both touch devices are Synaptics controllers speaking the standard RMI4
protocol over I2C (page select register 0xff). The in-tree Linux RMI4 driver
(`rmi4_i2c` with F01/F11/F12) handles both; no device-specific driver is needed.

## Touchscreen

| item | value |
|---|---|
| controller | Synaptics, RMI4 product id "57695-006" |
| bus | i2c-0 (`0x11c20000`), 400 kHz in stock, open-drain |
| address | 0x67 |
| interrupt | GPIO7 (EINT 7); stock uses falling edge, our DT uses level low |
| reset | GPIO6 is routed as reset, but stock never drives it (reset by the F01 command); not described in our DT |
| supply | MT6369 vtp, exactly 2.8 V (comes up at 1.8 V; stock sets 2.8 V on probe and resume) |
| startup delay | 100 ms after power-on |
| reporting function | F12, 2D touchscreen, 7-byte objects (no Z) |
| sensor range | about 0..3238 on both axes |
| display | 1440x1440 |

The factory touchscreen driver (GPL v2 module synaptics_tp.ko of Titan 2
V01.00.14) scales coordinates to 1440 pixels (`raw * 1440 / 3238`) and
applies two filters: it drops a contact exactly at x = 720, and it drops a
contact closer than 200 pixels to another contact in the same frame. Mainline
reports raw sensor coordinates with the correct declared range; userspace
scales. Neither filter is applied on mainline.

In suspend stock puts F01 to sleep, disables the interrupt and turns vtp off
(unless the wake gesture is enabled); mainline RMI4 behaves the same way
without the gesture.

Device tree:

```
touchscreen@67 {
	compatible = "syna,rmi4-i2c";
	reg = <0x67>;
	interrupts-extended = <&pio 7 IRQ_TYPE_LEVEL_LOW>;
	vdd-supply = <&mt6369_vtp>;
	syna,startup-delay-ms = <100>;
	#address-cells = <1>;
	#size-cells = <0>;
	rmi4-f01@1 { reg = <0x01>; };
	rmi4-f12@12 { reg = <0x12>; syna,sensor-type = <1>; };
};
```

If the controller does not respond after a cold start, add
`reset-gpios = <&pio 6 GPIO_ACTIVE_LOW>`. If the level-triggered EINT
misbehaves, fall back to `IRQ_TYPE_EDGE_FALLING`.

## Keyboard touchpad

The capacitive surface of the keyboard is a separate touch controller.

| item | value |
|---|---|
| controller | Synaptics **S1403** (firmware family DS5, RMI4 F11) |
| bus | i2c-2 (`0x11c22000`), 100 kHz |
| address | 0x20 |
| interrupt | GPIO75 (EINT 75); stock falling edge, our DT level low |
| reset | GPIO4, active low |
| power | load switch on GPIO59, high = on; no regulator |
| reported range | 1440 x 720 (from the sensor) |
| report rate | about 52 Hz, one contact |

Power-up sequence used by stock: reset high and power on, 20 ms, then reset
pulse low for 20 ms, then 120 ms before the first access. Stock additionally
pulses reset when it sees a contact with zero width in both axes; mainline
does not.

Device tree:

```
touchpad_vdd: regulator-touchpad-vdd {
	compatible = "regulator-fixed";
	regulator-name = "touchpad_vdd";
	gpio = <&pio 59 GPIO_ACTIVE_HIGH>;
	enable-active-high;
	startup-delay-us = <20000>;
};

touchpad@20 {
	compatible = "syna,rmi4-i2c";
	reg = <0x20>;
	interrupts-extended = <&pio 75 IRQ_TYPE_LEVEL_LOW>;
	reset-gpios = <&pio 4 GPIO_ACTIVE_LOW>;
	vdd-supply = <&touchpad_vdd>;
	syna,reset-delay-ms = <120>;
	#address-cells = <1>;
	#size-cells = <0>;
	rmi4-f01@1 { reg = <0x01>; };
	rmi4-f11@11 { reg = <0x11>; syna,sensor-type = <2>; };
	rmi4-f12@12 { reg = <0x12>; syna,sensor-type = <2>; };
};
```

The in-tree `rmi_i2c` transport did not handle a reset GPIO; our kernel adds
optional `reset-gpios` support. `syna,sensor-type = <2>` makes the device a
pointer (touchpad) for libinput, which also enables disable-while-typing with
the keyboard. Stock Android instead treats it as a direct-touch device and
turns gestures into cursor and scroll events in its input framework; use
`syna,sensor-type = <1>` to get the same direct-touch behaviour.

### Second-source touchpad

The stock tree also describes a Hynitron CST76xx touchpad at 0x5b on the same
GPIOs, used on other production batches. On the analysed phone it is not
present (the S1403 claims the GPIOs first). Its protocol uses 4-byte
big-endian register addresses with a 16-bit checksum over each report, which
the mainline `hynitron_cstxxx` driver does not support; not needed for this
device.

### Rear display touch

A Hynitron touch controller for the small rear display sits at 0x5a on i2c-2
(410 x 502). Not supported yet.

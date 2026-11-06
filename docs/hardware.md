# Hardware

[Overview](../README.md) · [Trackpad and input processing](trackpad.md)

## Boards and roles

| Component | Hardware / build target | Role |
| --- | --- | --- |
| Left half | nice!nano v2, nRF52840, Pro Micro footprint / `nice_nano_v2` | BLE keyboard peripheral with OLED |
| Right half | nice!nano v2, nRF52840, Pro Micro footprint / `nice_nano_v2` | BLE keyboard and trackpad peripheral |
| Dongle | Prospector hardware, XIAO BLE / `seeeduino_xiao_ble` | Central; USB or Bluetooth connection to the host |

The PCB is a true **five-column Corne**: 3×5 keys plus three thumbs per half,
**36 keys** total. The in-repository shields are `pones_corne_left`,
`pones_corne_right`, and `pones_corne_dongle`. The dongle additionally uses
YADS's vendored `dongle_screen` shield.

**The keyboard needs the dongle.** Neither half works as an independent host
keyboard or mouse with these images. Only the dongle enables
`CONFIG_ZMK_SPLIT_ROLE_CENTRAL`; both halves are peripherals. The central
reserves two split BLE peripherals and seven connection/bond slots: two halves
and five host profiles.

The firmware uses **ZMK v0.3.0**, including its upstream Zephyr/module manifest,
as declared in [the West manifest](../config/west.yml). The host keyboard layout
must be **US ANSI**: symbols and shortcut macros send HID keycodes, not Unicode
or layout-independent text.

The right half carries a circular Cirque GlidePoint/Pinnacle I²C trackpad.
Confirm the actual sensor model and wiring against the [trackpad reference](trackpad.md).
The nice!nano v2 has enough pins for both the matrix and trackpad; no controller
change is needed. These images target the nRF52840 controllers above, not
AVR/ATmega32U4 controllers.

Before installation, confirm the PCB pinout, controller orientation, and power
wiring. **Do not connect a powered TRRS cable between wireless halves.**

## Five-column matrix wiring

Each half scans four row GPIOs and **five column GPIOs**, with standard Corne
`col2row` diode wiring. There is no sixth scanned column or padded keymap.
The shield uses upstream Corne's five-column physical layout and one
36-position transform.

GPIO numbers below are **Pro Micro pad names**, not physical header indices.
The halves are viewed from above.

| Lines | Left | Right |
| --- | --- | --- |
| Rows 0–3 | D4, D5, D6, D7 | D4, D5, D6, D7 |
| Columns 0–4 | D20, D19, D18, D15, D14 | D14, D15, D18, D19, D20 |
| Thumb row 3 | Columns 2, 3, 4 | Columns 0, 1, 2 |

The right transform adds a five-column offset. Global key positions are 0–29
for the three letter rows and 30–35 for the thumbs. **D21 is not scanned.**
This is the standard Corne inner-column wiring assumption. If the actual
five-column PCB routes differently, change the shield row/column GPIO lists
before flashing. D1 is free in this matrix and carries the trackpad's
[data-ready signal](trackpad.md#wiring-and-sensor-settings).

The upstream reference is the [ZMK v0.3.0 Corne shield](https://github.com/zmkfirmware/zmk/tree/v0.3.0/app/boards/shields/corne).

Relevant definitions:

- [Shared layout and transform](../config/boards/shields/pones_corne/pones_corne.dtsi)
- [Matrix scanner and rows](../config/boards/shields/pones_corne/pones_corne_matrix.dtsi)
- [Left column GPIOs](../config/boards/shields/pones_corne/pones_corne_left.overlay)
- [Right column GPIOs and offset](../config/boards/shields/pones_corne/pones_corne_right.overlay)

The dongle uses an empty mock scanner with the same 36-position transform;
it generates no phantom local keys.

## Dongle display

Standard Prospector hardware combines a XIAO BLE with an **ST7789 240×280**
display, PWM backlight, and **APDS9960** ambient-light sensor.

The screen uses **YADS (`janpfischer/zmk-dongle-screen`)**, licensed under MIT.
It is a **vendored local snapshot dated 2026-11-06** in
[`vendor/zmk-dongle-screen`](../vendor/zmk-dongle-screen), with its
[MIT license](../vendor/zmk-dongle-screen/LICENSE) retained. Vendoring keeps the
screen implementation available locally with the ZMK v0.3 / Zephyr 3.5 / LVGL 8
stack it targets. YADS and `prospector-zmk-module` **cannot coexist**: both
register the ST7789 driver and LVGL glue.

A local patch in
[`layer_status.c`](../vendor/zmk-dongle-screen/boards/shields/dongle_screen/src/widgets/layer_status.c)
converts the active layer index to a layer ID before calling
`zmk_keymap_layer_name()`. The conversion is a no-op while layer reordering and
Studio are disabled.

YADS displays:

- Current output: USB or Bluetooth, including the Bluetooth profile.
- Active layer name, modifiers, and words per minute.
- Per-half battery levels.

It blanks after **600 seconds** idle. Automatic brightness uses the APDS9960
through `CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT=y`. If the physical build omits
that sensor, set this option to `n` in
[`config/pones_corne_dongle.conf`](../config/pones_corne_dongle.conf).
The display thread stack is explicitly set to 4096 bytes; layer-name redraws
exceeded a 2048-byte stack in recorded local use.

F22 toggles the display; F23 lowers brightness and F24 raises it. The keymap
does not bind these keys. **Caps Word has no screen indicator.**

## Half displays and input forwarding

The left half enables ZMK's built-in peripheral status screen on a stock Corne
**128×32 SSD1306 OLED**, `ssd1306@3c`, using the OLED-header I²C bus. The right
half's OLED remains disabled. Role-specific defaults are in
[`Kconfig.defconfig`](../config/boards/shields/pones_corne/Kconfig.defconfig).

All three builds declare input-split ID **0**. It is enabled on the right
physical source and dongle proxy, but disabled on the sensorless left: ZMK
v0.3 requires every enabled peripheral input-split node to have a physical
source device. Only the dongle's input listener emits host mouse reports.
See [trackpad input routing and tuning](trackpad.md#input-routing).

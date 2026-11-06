# Trackpad

[Overview](../README.md) · [Hardware and matrix wiring](hardware.md)

## Sensor identification

The right half uses a **Cirque GlidePoint/Pinnacle on I²C at `0x2a`**, with a
DR/data-ready line on D1. Recorded local UF2 inspection identified the device
as `glidepoint@2a`, driven by `cirque,pinnacle`.

The exact module remains an assumption: **40 mm TM040040, Gen2 Pinnacle
`1CA027`, 12-pin FFC, configured for I²C**. Confirm the ASIC and full part
number on the physical module before installation. References:

- [Keycapsss module and specifications](https://keycapsss.com/GlidePoint-Cirque-Trackpad-TM040040-TM035035/KC10079-40-C)
- [Corne + Cirque build log](https://beekeeb.com/cirque-trackpad-i2c-on-corne-keyboard/): demonstrates the OLED-header I²C wiring approach with a 23 mm Pinnacle, not this build's exact PCB or trackpad model.
- [TM040040 specification, pages 8–9](https://keycapsss.com/media/43/54/77/1758718807/tm040040_spi-i2c-pinntrackpad_spec1-2-1223705.pdf?ts=1758718807): factory I²C parts are `TM040040-2023-xxx`; SPI parts are `TM040040-2024-xxx` and need the documented I²C-mode conversion. Follow the actual module documentation rather than guessing resistor changes.

**Do not substitute a Gen6/Gen6s “Timp” circle pad merely because its diameter
or model prefix matches.** The [current Cirque circle-pad range](https://www.cirque.com/glidepoint-circle-trackpads)
includes a different controller/interface. The Pinnacle driver does not imply
support for it.

The sensor requires **DR/data-ready**, not just a four-wire I²C adapter. Break
out J1 pin 4 (DR), along with pins 9 (SCL), 10 (SDA), 11 (GND), and 12 (3.3 V).
**Verify FFC contact orientation before powering the module.**

## Input routing

ZMK v0.3.0 supports peripheral input forwarding but does not bundle the sensor
driver. [The West manifest](../config/west.yml) includes the
[Cirque input module](https://github.com/petejohanson/cirque-input-module).
Only the right shield instantiates and enables the physical sensor.

All three builds share **input-split ID 0**:

1. The right half reads the sensor and forwards events over split BLE.
2. The dongle proxy receives those events.
3. The **dongle-only input listener** applies processors and emits host mouse reports.

The sensorless left has neither an enabled input-split source nor a host
listener. Its shared input-split declaration remains disabled.

## Central input processors

The [dongle overlay](../config/boards/shields/pones_corne/pones_corne_dongle.overlay)
uses stock [ZMK input processors](https://zmk.dev/docs/keymaps/input-processors).
Processors are configured in that overlay rather than behind a keymap
`CONFIG_*` guard: Kconfig role symbols are not available during devicetree
preprocessing in this build.

| Mode | Layer index | Processor | Effect |
| --- | --- | --- | --- |
| Cursor | Base chain | `&zip_xy_scaler 3 1` | ×3 XY movement, retaining fractional remainders |
| MOUSE | 5 | `&zip_xy_to_scroll_mapper` | Unscaled scrolling: X becomes horizontal wheel; Y becomes vertical wheel |
| PAD x1 | 7 | `&zip_xy_scaler 1 1` | Unscaled cursor movement |
| PAD x6 | 8 | `&zip_xy_scaler 6 1` | ×6 cursor movement |

The layer indices must match [the shared keymap](../config/pones_corne.keymap).
Update both the keymap and overlay if reordering layers. SYS+D toggles PAD x1;
SYS+E toggles PAD x6. The first matching override wins: MOUSE scrolling takes
precedence over both speed layers, and PAD x1 wins if both speed layers are on.

Layer overrides **replace** the base processor chain in ZMK v0.3.0. To tune
scroll speed separately, prepend a scaler to the MOUSE override. Do not add
`process-next`: that would also run the base ×3 scaler. For cursor scaling,
lower the numerator or raise the denominator for slower movement; `1 1` is
unscaled. Keep the denominator positive.

Taps and button events pass through the XY processors unchanged. There is
**no `zip_temp_layer`**: automatic MOUSE activation would turn ordinary pad
movement into scrolling and alter the layer-tap interaction.

The dongle enables `CONFIG_ZMK_POINTING_SMOOTH_SCROLLING=y`, allowing HID
resolution multipliers on compatible hosts. Actual high-resolution scrolling
behavior depends on host support; verify both the host and KVM.

**Processor tuning only requires rebuilding and reflashing the dongle.**
Orientation, thresholds, and ADC sensitivity are right-overlay settings and
require rebuilding and reflashing the right half.

## Wiring and sensor settings

A recorded local validation decoded `struct pinnacle_config` from
`backups/2026-11-05/right-CURRENT.UF2` (an untracked local backup).
The recorded inspection identified the bus, GPIOs, mounting, sensitivity, sleep,
and contact thresholds listed below. Applying the same decoder to a repository
build reproduced the overlay values exactly. This is **historical local
validation**, not a fresh hardware or build check.

| Trackpad signal | Right controller connection |
| --- | --- |
| SDA | Pro Micro **D2 = nRF P0.17** |
| SCL | Pro Micro **D3 = nRF P0.20** |
| DR | Pro Micro **D1 = nRF P0.06**, active high |

D1 is the stock Corne serial pad. The [five-column matrix](hardware.md#five-column-matrix-wiring)
does not use it, and `pro_micro_serial` / `uart0` is disabled.
Supply the sensor from **regulated 3.3 V**, not RAW, battery positive, or USB
5 V. Keep **4.7 kΩ pull-ups on each I²C line** if the module does not provide
them.

The [right overlay](../config/boards/shields/pones_corne/pones_corne_right.overlay)
configures:

| Setting | Value |
| --- | --- |
| Bus | `&pro_micro_i2c` (`i2c0`), **100 kHz** |
| Address | **`0x2a`** |
| Data ready | `dr-gpios = <&pro_micro 1 GPIO_ACTIVE_HIGH>` |
| ADC sensitivity | **`"2x"`**; driver supports `"1x"` through `"4x"` |
| Mounting | No rotation; **`y-invert`** |
| X contact threshold | `x-axis-z-min = <5>` |
| Y contact threshold | `y-axis-z-min = <4>` |
| Sensor sleep | Enabled with `sleep` |

ADC sensitivity is **not cursor acceleration**. Native taps and secondary
taps remain enabled; the thumb buttons also provide explicit clicks.

The bus pin mappings are inherited from the nice!nano board configuration;
the right overlay does **not** currently define `i2c0_default` or `i2c0_sleep`.
If changing bus pins, add the appropriate pinctrl overrides for those states
in the right overlay. Changing only a connector label does not reroute I²C.

The [ZMK v0.3.0 nice!nano pinctrl](https://github.com/zmkfirmware/zmk/blob/v0.3.0/app/boards/arm/nice_nano/nice_nano-pinctrl.dtsi)
and [Pro Micro connector map](https://github.com/zmkfirmware/zmk/blob/v0.3.0/app/boards/arm/nice_nano/arduino_pro_micro_pins.dtsi)
document the nRF mappings above.

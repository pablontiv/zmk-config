# Validation and known limitations

## Recorded local builds

These results were recorded during development and are **not rerun per commit**. Revalidate after changing the keymap or modules. All three images built locally with ZMK v0.3.0, the pinned Cirque module, the vendored YADS screen, **Zephyr SDK 0.16.8, west 1.5.0, CMake 3.28.4, and Python 3.13.5**.

| Shield | Board | Flash used / available | RAM used / 256 KiB | UF2 bytes |
| --- | --- | --- | --- | --- |
| `pones_corne_left` | `nice_nano_v2` | 170864 B / 792 KiB (21.07%) | 33868 B (12.92%) | 342016 |
| `pones_corne_right` | `nice_nano_v2` | 175588 B / 792 KiB (21.65%) | 34100 B (13.01%) | 351232 |
| `pones_corne_dongle` + `dongle_screen` (YADS) | `seeeduino_xiao_ble` | 423128 B / 788 KiB (52.44%) | 212548 B (81.08%) | 846336 |

The display-heavy dongle leaves **49596 B** outside linked RAM allocations. Runtime stack/heap headroom has not been measured on hardware. Both `settings_reset` images also built:

| Board | Flash used | RAM used | UF2 bytes |
| --- | --- | --- | --- |
| nice!nano | 46188 B | 11552 B | 92672 |
| XIAO | 49512 B | 11640 B | 99328 |

Firmware is under the ignored `build/` directories listed in [build and flash](build-and-flash.md).

## Recorded structural and runtime checks

Validation reused an external west/toolchain workspace, passing this checkout's `config` via `ZMK_CONFIG` and the vendored YADS module via `ZMK_EXTRA_MODULES`. The Cirque module is discovered through `west update`.

The recorded compiled-devicetree check covered five scanned columns and four rows per half, 36 transform entries, 36 physical keys, and 36 bindings on each of six layers. That **six-layer count describes the recorded check, not the current keymap**. The current [`config/pones_corne.keymap`](../config/pones_corne.keymap) defines **nine layers**, each with 36 bindings: QWERTY, NUMBER, SYMBOL, MOVE, NAV, MOUSE, SYS, PAD x1, and PAD x6. This documentation update does not claim a fresh compiled-devicetree check of all nine layers.

The recorded devicetree checks also covered the dongle's empty mock scanner and processor chains, right-only Cirque, dongle-only listener, and split ID 0 on all three devices. Generated Kconfig confirmed two peripheral roles, the central role, two split peers, seven connection/bond slots, dongle display and smooth scrolling, and right-side Pinnacle.

There is **no native test suite** in this configuration repository. A throwaway `native_posix_64` runtime included the actual dongle overlay and shared keymap: **12 assertions passed**, producing **ten captured mouse reports**. It exercised the production central split-event handler → input-split proxy → listener, covering:

- Half-speed XY and retained fractional motion.
- The right MOUSE thumb's 220 ms hold/release and two-axis scrolling while held.
- Pad button press/release passthrough and left-side fallback right-click/release.
- Cursor restoration and NAV cursor movement.

Logical key events replaced matrix scanning; injected split events replaced BLE delivery; mouse reports were intercepted at the endpoint boundary. BLE host-profile switching, KVM acceptance, display appearance, physical scanning, and the Cirque driver's communication with silicon were **not simulated or certified** by this smoke. No smoke scaffolding is committed.

## Build warnings

Recorded non-fatal upstream warnings include:

- Deprecated `config/boards` discovery and `NRF_STORE_REBOOT_TYPE_GPREGRET`.
- Peripheral USB defaults.
- XIAO devicetree duplicate-address and deprecated-label warnings.
- An `-Wint-conversion` warning from ZMK's own `central_bas_proxy.c`.
- Empty-keymap array warnings from stock `settings_reset`.

All five recorded firmware builds completed; no module/API build blocker was found in those builds. These are recorded results, not checks run for this documentation update.

## Configured functionality

The [layout](layout.md) provides six everyday layers in 36-key order, eight home-row mods, five everyday thumb layer-taps plus Delete/SYS, F+J Caps Word, eight symbol behaviors, seven Herdr macros, two KVM macros, media/navigation, right-peripheral physical pointing, fallback mouse movement/scroll/buttons, Caps Lock, Insert, and per-device bootloader/reset controls. SYS groups device, Bluetooth, and output controls; two fully transparent toggle layers control trackpad speed. Together, these make the current nine-layer keymap.

## Known limitations

- Delimiter dances retain single/double taps but do not type `{}`, `()`, `<>`, or `[]` pairs on hold. Type pairs manually.
- Hold symbols are not one-shot: outputs remain pressed until release and may auto-repeat. Physically held modifiers are not cleared by the macros. Tap-dance, hold-tap, and Caps Word use stock ZMK semantics and timing.
- The left OLED uses ZMK's stock peripheral status screen, without custom logos or font. The dongle uses YADS's stock status screen.
- No console keystroke logger or debug-toggle action is configured; the debug-toggle key position is blocked.
- Mouse behavior uses the configured ZMK acceleration and input processors. There is no custom USB polling, debounce, or EEPROM implementation, no RGB matrix animations, and no in-repository runtime C or driver fork.
- Trackpad DR wiring, orientation, and sensitivity are configured, but real I²C communication with this build still requires hardware validation, as does the exact module part number.
- Split pairing, trackpad battery/power behavior, BLE/USB host reports, and physical KVM acceptance require the actual keyboard. A local build cannot certify these.
- The following options have been reviewed but are **not enabled**; evaluate only after hardware soak or measurement:
  - `CONFIG_ZMK_SLEEP` with an idle timeout on the halves; the matrix already has `wakeup-source`.
  - Hold-tap `require-prior-idle-ms` / `quick-tap-ms` for home-row mods.
  - `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN` only if host links drop.
  - LVGL pool/font trimming only if the dongle UI misbehaves; linked dongle RAM use is approximately 81%.
  - ZMK Studio on the dongle.
  - Moving to ZMK main / Zephyr 4.1.

## Licensing

This repository is MIT-licensed; see [LICENSE](../LICENSE).

Vendored code in [`vendor/zmk-dongle-screen`](../vendor/zmk-dongle-screen/) retains its own terms. The module is MIT-licensed; see its [LICENSE](../vendor/zmk-dongle-screen/LICENSE). Files with their own SPDX headers remain under those licenses, including the Apache-2.0 ST7789 display driver.

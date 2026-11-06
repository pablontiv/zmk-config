# Pones five-column Corne + Cirque + Prospector hardware (YADS screen) — ZMK

Personal ZMK configuration for a 36-key, five-column Corne with a right-hand
Cirque trackpad and a Prospector dongle running the YADS screen firmware,
ported from the [QMK `crkbd/rev1:pones`](https://github.com/pablontiv/qmk_userspace/tree/main/keyboards/crkbd/rev1/keymaps/pones)
keymap. Firmware is built locally; this repository has no GitHub Actions build
workflow.

## Hardware and firmware

- **Boards:** two `nice_nano_v2` controllers (nRF52840, Pro Micro footprint)
  plus a **Prospector BT dongle**, built as `seeeduino_xiao_ble`.
- **PCB:** true **five-column Corne**, 3×5 + 3 thumbs per half, **36 keys**.
- **Shields:** in-repo `pones_corne_left`, `pones_corne_right`, and
  `pones_corne_dongle`; the dongle also uses the vendored YADS `dongle_screen`.
- **Pointing:** circular Cirque GlidePoint/Pinnacle I²C trackpad on the right.
  The model and wiring assumptions below must be checked against the actual PCB.
- **Central:** Prospector-hardware dongle; it connects to the host over USB or Bluetooth.
  Both keyboard halves are BLE peripherals. **The keyboard needs the dongle**:
  neither half works as an independent host keyboard/mouse with these images.
- **ZMK:** v0.3.0, pinned to `edf5c0814fd3ea202e43aad2d68fd32e882a518c`
  in `config/west.yml`, including its upstream Zephyr/module manifest.
- **Host layout:** US ANSI. Symbols and shortcut macros are HID keycodes,
  not Unicode or layout-independent text.

The custom shield scans only five column GPIOs per half and uses one 36-position
transform; there is no sixth matrix column or padded keymap. It reuses upstream
Corne's five-column physical layout and standard Corne row/inner-column wiring.
The nice!nano v2 has enough pins for this matrix and the trackpad; no controller
change is required. This is **not** firmware for AVR/ATmega32U4 QMK controllers.
Confirm PCB pinout, controller orientation, and power wiring before installation;
do not connect a powered TRRS cable between wireless halves.

### Dongle central (Prospector hardware, YADS firmware)

The dongle hardware is a standard Prospector (XIAO BLE, ST7789 240×280, PWM
backlight, APDS9960), built as `seeeduino_xiao_ble`. Its screen firmware is
[YADS `janpfischer/zmk-dongle-screen`](https://github.com/janpfischer/zmk-dongle-screen)
(MIT), **vendored** under `vendor/zmk-dongle-screen` because the fetched copy
carried no Git revision to pin (snapshot fetched 2026-11-06). It targets the
same ZMK v0.3 / Zephyr 3.5 / LVGL 8 stack and replaces the former
`prospector-zmk-module`; the two cannot coexist (both register the ST7789
driver and LVGL glue). Local change to the vendored copy:
`src/widgets/layer_status.c` converts the active layer index to an id before
`zmk_keymap_layer_name()` (a no-op while layer reordering/Studio stay off).

The dongle uses an empty mock scanner and the same 36-position transform as
the halves; no phantom keys are generated. `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`
only there; both halves resolve it to `n`. The central reserves two split BLE
peripherals and seven connection/bond slots (two halves plus five host
profiles). Input-split ID 0 is declared on all three, enabled on the right
source and dongle proxy, and disabled on the sensorless left: v0.3 requires
every enabled peripheral input-split node to have a physical source device.

YADS shows output (USB/BLE + profile), active layer name, modifiers, WPM and
per-half battery, blanks after 600 s idle, and drives brightness from the
APDS9960 (`CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT=y`). F22/F23/F24 adjust/toggle
brightness; the keymap does not bind them. Caps Word has no indicator (the
old Prospector screen had one). If the actual build omits the light sensor,
set `CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT=n` in `config/pones_corne_dongle.conf`.
The left half drives the stock Corne 128×32 SSD1306 OLED (`ssd1306@3c` on the
OLED-header I²C bus, as in the previous working left firmware) with ZMK's
built-in peripheral status screen; the right half's OLED stays disabled.

## Layout and behavior

`config/pones_corne.keymap` keeps QMK's physical order and layer order:
QWERTY, NUMBER, SYMBOL, MOVE, NAV, MOUSE, plus ZMK-only SYS (6) and transparent
trackpad-speed toggle layers PAD x1 (7) / PAD x6 (8).
`config/pones.dtsi` contains
stock ZMK behaviors/macros; `config/pones_corne.conf` enables pointing and disables
underglow. Shield definitions and half-display defaults live in
`config/boards/shields/pones_corne/`; central settings are in
`config/pones_corne_dongle.conf`.

Base, viewed from above:

```text
Q  W  E  R  T       Y  U  I  O  P
A  S  D  F  G       H  J  K  L  ;
Z  X  C  V  B       N  M  ,  .  '
    Esc Tab Space   Enter Bspc Del
```

Home-row mods, **tap / hold**:

| Key | Hold |
| --- | --- |
| A | Left GUI |
| S | Left Alt |
| D | Left Ctrl |
| F | Left Shift |
| J | Right Shift |
| K | Right Ctrl |
| L | Left Alt (intentional, matches QMK) |
| ; | Right GUI |

Thumbs, left to right:

| Tap | Hold |
| --- | --- |
| Escape | NAV |
| Tab | SYMBOL |
| Space | NUMBER |
| Enter | MOVE |
| Backspace | MOUSE |
| Delete | SYS |

Home-row mods and layer-taps use **tap-preferred, 220 ms**. Interrupting with
another key does not force a hold: hold past 220 ms to use the modifier/layer.
This follows the QMK configuration's intent without adding permissive hold or
bilateral rules; the two firmware event engines are not timing-identical.
Hold the thumb before using a layer key. Releasing it returns to base even
though its position is blocked on the active layer.

**F+J within 30 ms on base** (zero-based positions **13 + 16**) toggles stock
ZMK Caps Word. It uses ZMK's standard continuation rules (no idle timeout in
this pinned release), not a custom QMK emulation.

### Layers

All unspecified keys are `&none`, **not transparent**. This preserves QMK's
`XXXXXXX` behavior: base letters, modifiers, and thumb taps do not leak through.

| Layer | Left hand | Right hand / thumbs |
| --- | --- | --- |
| NUMBER | F1–F4 / F5–F8 / F9–F12 | 7 8 9 `*` / `=` 4 5 6 `+` / 1 2 3 `/`; right thumbs `- . 0` |
| SYMBOL | Home row: Print Screen, Ctrl+C, Ctrl+V, Ctrl+Shift+V | Grave, ampersand/pipe, hash, slash/backslash, caret; `@` and delimiters; tilde, minus/underscore, dollar, percent, question/exclamation |
| MOVE | Herdr commands, Ctrl+Shift+Tab / Ctrl+Tab, KVM previous / next | Media row |
| NAV | Home row: GUI, Alt, Ctrl, Shift | Home / Up / End; Page Up / Left / Down / Right / Page Down; Alt+Left / Alt+Right |
| MOUSE | Trackpad is primary; E = up, S/D/F = left/down/right and T/G = scroll up/down are keyboard fallback; Tab/Space thumbs = right/left click | Y-position = Insert, I = Caps Lock; QMK debug-toggle position is blocked |
| SYS | Q = left bootloader, W = left reset, E / D = toggle trackpad x6 / x1; Z = dongle bootloader (single-key combo); A/S = BT profile 0/1, C = clear selected host bond, F/G = prefer USB/BLE | O = right reset, P = right bootloader |

The physical trackpad moves the cursor on base and all non-MOUSE layers.
Hold Backspace/MOUSE past the layer-tap term to turn pad XY into horizontal/
vertical scrolling; release it to return to cursor movement. Native pad taps
and thumb mouse buttons remain available. Keyboard movement/scroll remain
fallbacks before trackpad wiring/tuning is complete. Keyboard movement uses
linear acceleration over 300 ms (ZMK stock and factory ramp) to 1200 units/s
(2x ZMK's stock 600); this is separate from the physical pad's input
processor tuning below.

### Symbols: tap dance and hold-tap

On SYMBOL:

| Base position | Single tap | Double tap | Hold |
| --- | --- | --- | --- |
| J | `{` | `}` | `{` (stock dance, no pair) |
| K | `(` | `)` | `(` (stock dance, no pair) |
| L | `<` | `>` | `<` (stock dance, no pair) |
| ; | `[` | `]` | `[` (stock dance, no pair) |
| U | `&` | — | `\|` |
| O | `/` | — | `\` |
| M | `-` | — | `_` |
| ' | `?` | — | `!` |

Delimiter dances use a 180 ms tapping term. A different key ends the dance
early; holding a dance selects the current tap count. The other four symbols
are ordinary tap-preferred hold-taps, also 180 ms, not multi-tap dances.
Hold outputs remain pressed until release and may auto-repeat on the host;
QMK's handlers emitted one-shot symbols.

### Macros and KVM

All MOVE macro positions are retained using stock ZMK macros:

| Base position | Output |
| --- | --- |
| W / S | Herdr previous / next space: Ctrl+B, then `,` / `.` |
| E / D | Herdr previous / next tab: Ctrl+B, then `p` / `n` |
| Z | Herdr horizontal split: Ctrl+B, then `-` |
| X | Herdr new space: Ctrl+B, then uppercase `N` |
| C | Herdr new tab: Ctrl+B, then `c` |
| T / G | KVM previous / next: Right Ctrl, Right Ctrl, Page Up / Page Down |

Herdr sequences use 10 ms presses and a 10 ms gap after Ctrl+B. KVM uses
10 ms presses and 50 ms gaps between released keys. Modifiers are released
between steps. Invoke with other modifiers released; these macros do not
clear physically held modifiers. The Herdr application must recognize those
bindings. KVM sequences target an external KVM's hotkey protocol, **not ZMK
Bluetooth profile switching**. Connect the **Prospector dongle's USB** to that
KVM; recognition depends on the KVM and must be checked on hardware.

SYS (hold Delete) groups device/system controls away from everyday layers:

| Base position | Binding | Effect |
| --- | --- | --- |
| Q / P | `&bootloader` | UF2 bootloader on the **half whose key is pressed** (left / right) |
| W / O | `&sys_reset` | Reboot the left / right half |
| Z (single-key combo) | `&bootloader` | Combos run on the central: UF2 bootloader on the **dongle** |
| A / S | `&bt BT_SEL 0` / `&bt BT_SEL 1` | Select dongle host profile 0 / 1 |
| C | `&bt BT_CLR` | Clear **selected host profile's** bond; not split bonds |
| F / G | `&out OUT_USB` / `&out OUT_BLE` | Prefer USB / Bluetooth host output |

Hold Delete past 220 ms to reach SYS; a quick tap still sends Delete, but
holding Delete no longer auto-repeats. Keyboard-triggered bootloader/reset on a
half requires that half to be linked to the dongle (the layer state lives on
the central); otherwise use the hardware reset button.

Select USB before KVM macros; use BLE output plus a profile selection for a
Bluetooth host. Output preference applies when that transport is ready; ZMK
can fall back to the available transport. Profile selection does not change
the KVM's physical port or the keyboard-halves' split pairing. Clear a profile
only when intending to forget/re-pair its host.

## Right-hand circular trackpad

**Identified from the previous working firmware:** a Cirque GlidePoint
**Pinnacle on I²C at `0x2a`** with a DR line on D1 (see "Wiring and tuning").
The module/part number below is still an assumption: a **40 mm TM040040,
Gen2 Pinnacle `1CA027`, 12-pin FFC, configured for I²C**. This is the conventional
keyboard module sold with Corne mounting options; see the
[Keycapsss module/specifications](https://keycapsss.com/GlidePoint-Cirque-Trackpad-TM040040-TM035035/KC10079-40-C)
and [Corne + Cirque build log](https://beekeeb.com/cirque-trackpad-i2c-on-corne-keyboard/).
The latter uses a 23 mm Pinnacle and establishes the OLED-header I²C wiring
approach, not this build's exact PCB or trackpad model.
The [original TM040040 specification, pp. 8–9](https://keycapsss.com/media/43/54/77/1758718807/tm040040_spi-i2c-pinntrackpad_spec1-2-1223705.pdf?ts=1758718807)
identifies factory I²C parts as `TM040040-2023-xxx` and SPI parts as
`TM040040-2024-xxx`; the latter need the documented I²C-mode conversion.
Use the actual module's documentation rather than guessing resistor changes.
The firmware requires **DR/data-ready**, not just a four-wire I²C adapter:
break out J1 pin 4 (DR) in addition to pins 9 (SCL), 10 (SDA), 11 (GND),
and 12 (3.3 V). Verify FFC contact orientation before powering it.

**Do not substitute a newer Gen6/Gen6s “Timp” circle pad** merely because its
diameter/model prefix matches. Cirque's
[current circle-pad range](https://www.cirque.com/glidepoint-circle-trackpads)
includes a different controller/interface; the Pinnacle driver is not a
claim of support for that hardware. Check the ASIC and full part number first.

Pinned ZMK v0.3.0 already supports peripheral input forwarding, but does not
bundle this driver. `config/west.yml` retains
[`petejohanson/cirque-input-module`](https://github.com/petejohanson/cirque-input-module/tree/0de55f36bc720b5be3d8880dc856d4d78baf5214)
at `0de55f36bc720b5be3d8880dc856d4d78baf5214`.
Only the right shield instantiates/enables the physical sensor; all three builds
share input-split ID **0**. The right forwards sensor events over split BLE;
the **dongle's** input listener applies processors and emits host mouse reports.
The left has no pointing hardware or host listener.

### Central input processors

`config/boards/shields/pones_corne/pones_corne_dongle.overlay` configures the
**dongle-only** listener with stock
[ZMK input processors](https://zmk.dev/docs/keymaps/input-processors).
The MOUSE override uses layer index 5, matching `MOUSE` in the shared keymap;
update both if reordering layers. Processors live in the dongle overlay rather
than a keymap `CONFIG_*` guard because Kconfig role symbols are not available
at devicetree preprocessing time in this pinned build.

- Cursor: `&zip_xy_scaler 3 1`, ×3 XY with fractional remainders, matching
  the base multiplier of the previous working firmware. Lower N or raise D
  for slower movement; `1 1` is unscaled. Keep D positive.
- MOUSE (layer 5) override: `&zip_xy_to_scroll_mapper` only, unscaled scroll as
  in the previous firmware. X becomes horizontal wheel, Y vertical wheel.
  Layer overrides **replace** the base chain in pinned ZMK; prepend a scaler
  here to tune scroll speed separately. Do not add `process-next`, which
  would also run the base scaler.
- Taps/button events pass through these XY processors unchanged.
- PAD x1 (layer 7) / PAD x6 (layer 8): `&zip_xy_scaler 1 1` / `6 1`, the
  factory firmware's "DPI Slow" / "DPI Fast" layers. Toggle with SYS+D / SYS+E;
  if both are on, x1 wins. MOUSE scroll wins over both (first matching override).
- `CONFIG_ZMK_POINTING_SMOOTH_SCROLLING=y` on central enables HID resolution
  multipliers for compatible hosts. Host support determines the observed
  high-resolution scrolling behavior; verify both the host and KVM.
- **No `zip_temp_layer`:** automatic MOUSE activation would turn ordinary pad
  movement into scrolling and alter the existing layer-tap interaction.

Only rebuild/reflash the dongle for processor tuning. Sensor orientation,
thresholds, and ADC sensitivity remain right-overlay settings and require
rebuilding/reflashing the right half.

### Wiring and tuning

Every sensor setting below was **extracted from the right half's previous
working firmware** (`backups/2026-11-05/right-CURRENT.UF2`, device
`glidepoint@2a`, driver `cirque,pinnacle`). Its `struct pinnacle_config` was
decoded from the UF2 payload. The same decoder, run on this repository's build,
reproduces this overlay's values exactly. The previous firmware was built from
the same Zephyr revision (`dacab4875df7`).

| Trackpad signal | Right controller connection (from previous firmware) |
| --- | --- |
| SDA | Pro Micro **D2 = nRF P0.17** |
| SCL | Pro Micro **D3 = nRF P0.20** |
| DR | Pro Micro **D1 = nRF P0.06**, active high |

D1 is the stock Corne serial pad. It is free here because the matrix does not
use it and `pro_micro_serial`/`uart0` is disabled. If rebuilding the hardware,
supply the pad from **regulated 3.3 V**, not RAW/battery+/USB 5 V, and keep the
I²C pull-ups (4.7 kΩ per line) if the module does not provide them.

All sensor settings are together in
`config/boards/shields/pones_corne/pones_corne_right.overlay`:

- Bus: `&pro_micro_i2c` (`i2c0`), **100 kHz**, address **0x2a**.
- DR: `dr-gpios = <&pro_micro 1 GPIO_ACTIVE_HIGH>`.
- Sensitivity: **`"2x"`**; driver supports `"1x"` through `"4x"` ADC sensitivity.
  This is not cursor acceleration.
- Mounting: no rotation, **`y-invert`**.
- Contact thresholds: `x-axis-z-min = <5>`, `y-axis-z-min = <4>`.
  Native taps/secondary taps remain enabled; thumb buttons provide explicit
  clicks. Pinnacle firmware **`sleep`** is enabled, as before.

For different bus pins, adjust `i2c0_default` and `i2c0_sleep` pinctrl in that
right overlay; changing only the connector label does not reroute I²C.
Pinned [nice!nano pinctrl](https://github.com/zmkfirmware/zmk/blob/edf5c0814fd3ea202e43aad2d68fd32e882a518c/app/boards/arm/nice_nano/nice_nano-pinctrl.dtsi)
and [connector map](https://github.com/zmkfirmware/zmk/blob/edf5c0814fd3ea202e43aad2d68fd32e882a518c/app/boards/arm/nice_nano/arduino_pro_micro_pins.dtsi)
establish the nRF mappings above.

### Five-column matrix wiring assumption

Standard Corne `col2row` wiring, with four row lines and **five** column lines
per half. GPIO numbers below are Pro Micro pad names, not physical header indices:

| Lines | Left, viewed from above | Right, viewed from above |
| --- | --- | --- |
| Rows 0–3 | D4, D5, D6, D7 | D4, D5, D6, D7 |
| Columns 0–4 | D20, D19, D18, D15, D14 | D14, D15, D18, D19, D20 |
| Thumb row 3 | Columns 2, 3, 4 | Columns 0, 1, 2 |

The right transform adds a five-column offset; global key positions are
0–29 for the three letter rows and 30–35 for the thumbs. D21 is not scanned.
This follows the [pinned upstream Corne inner-column wiring](https://github.com/zmkfirmware/zmk/tree/edf5c0814fd3ea202e43aad2d68fd32e882a518c/app/boards/shields/corne);
if the actual five-column PCB routes differently, change the shield row/column
GPIO lists before flashing. D1 is free in this wiring plan and carries DR.

## Build locally with west

No cloud build or CI setup is required. Follow the upstream
[v0.3 native toolchain setup](https://v0-3-branch.zmk.dev/docs/development/local-toolchain/setup/native)
for host dependencies and the Zephyr SDK compatible with ZMK v0.3.0
(SDK 0.16.8). On Linux this includes CMake, Ninja, device-tree-compiler,
Python with venv support, and SDK host tools. Use a supported Python version
for that Zephyr release (Python 3.11 is a conservative choice).

From this checkout, initialize it as a west workspace once (it must not be
inside another west workspace). Dependencies/build products stay ignored:

```sh
export PONES_CONFIG="$(pwd)"

python3 -m venv "$HOME/.venvs/zmk-pones"
. "$HOME/.venvs/zmk-pones/bin/activate"
python -m pip install west==1.5.0 cmake==3.28.4 ninja
west init -l config
west update
west zephyr-export
python -m pip install -r zephyr/scripts/requirements.txt
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
export ZEPHYR_SDK_INSTALL_DIR="$HOME/zephyr-sdk-0.16.8"
```

Set `ZEPHYR_SDK_INSTALL_DIR` to your actual SDK installation. On subsequent
shells, reactivate the venv, set `PONES_CONFIG` and the two toolchain variables
above, and enter this checkout. `west update` honors this repo's pinned manifest; after
pulling changes to this repo, run it again if the manifest changed.

Build **all three** targets, using separate build directories:

```sh
west build -p always -s zmk/app -b nice_nano_v2 -d build/pones_corne_left -- \
  -DSHIELD=pones_corne_left -DZMK_CONFIG="$PONES_CONFIG/config"
west build -p always -s zmk/app -b nice_nano_v2 -d build/pones_corne_right -- \
  -DSHIELD=pones_corne_right -DZMK_CONFIG="$PONES_CONFIG/config"
west build -p always -s zmk/app -b seeeduino_xiao_ble -d build/pones_corne_dongle -- \
  -DSHIELD="pones_corne_dongle dongle_screen" \
  -DZMK_CONFIG="$PONES_CONFIG/config" \
  -DZMK_EXTRA_MODULES="$PONES_CONFIG/vendor/zmk-dongle-screen"
```

Artifacts are `build/pones_corne_{left,right,dongle}/zephyr/zmk.uf2`
(three separate directories). Keep firmware/build products out of Git.

### Flash, reset old bonds, and pair

Flashing is scripted by the project skill
[`.agents/skills/zmk-flash`](.agents/skills/zmk-flash/SKILL.md). From the repo
root, with Docker (`zmkfirmware/zmk-build-arm:3.5`) and a west workspace in
`ZMK_WS` (default `.zmk/`, gitignored):

```sh
S=.agents/skills/zmk-flash/scripts/zmk-flash
$S all full      # pull, west update (pinned), build, settings_reset + flash all three
$S all halves    # rebuild and flash left then right, keeping bonds
$S mark-good     # after testing: mark the session and push tag flash/<date>-<sha>
```

The script prints an `ACTION:` line for each physical step (bootloader entry:
SYS+Z dongle, SYS+Q left, SYS+P right, or double-tap reset). Before writing,
it identifies each bootloader drive by Board-ID and by comparing its
`CURRENT.UF2` with recorded images, then saves that `CURRENT.UF2` under the
gitignored `backups/` directory. Use `full` (`--reset`) whenever the
central/peripheral roles or bonds change: it flashes `settings_reset` before
each normal image, which erases bonds and settings, not only the firmware.

1. Run the plan and follow its `ACTION:` lines. Keep the halves powered off
   after flashing until their pairing turn.
2. Plug the dongle into the host/KVM. Power **left only** and let it pair first;
   confirm the dongle screen shows its battery before continuing.
   Then power **right** and let it pair second, so peripheral slot 0 is left
   and slot 1 is right on the dongle's per-half battery widget.
3. USB host reports now come from the dongle. For Bluetooth, hold Delete past
   220 ms to reach SYS, press A or S (profile 0 or 1), then G (BLE output);
   pair **Pones Corne** in the host UI. Forget any stale host bond first to
   refresh the changed HID descriptor. SYS+F selects USB preference for the KVM.
4. Verify 36 keys, thumb taps/holds, home-row mods, F+J Caps Word, base cursor,
   held MOUSE two-axis pad scrolling, pad taps, thumb buttons, keyboard mouse
   fallbacks, dongle display, and application/KVM macros. USB on a half is
   flashing/power only. Then run `mark-good`.

If pairing fails, follow upstream
[split connection troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues)
and rerun `$S flash full` before pairing left then right.

## Local validation

These results were recorded during development and are not re-run per commit;
re-validate after changing the keymap or modules. All three images built
locally with the pinned ZMK and Cirque modules, the vendored YADS screen,
Zephyr SDK 0.16.8, west 1.5.0, CMake 3.28.4, and Python 3.13.5:

| Shield | Board | Flash used / available | RAM used / 256 KiB | UF2 bytes |
| --- | --- | --- | --- | --- |
| `pones_corne_left` | `nice_nano_v2` | 170864 B / 792 KiB (21.07%) | 33868 B (12.92%) | 342016 |
| `pones_corne_right` | `nice_nano_v2` | 175588 B / 792 KiB (21.65%) | 34100 B (13.01%) | 351232 |
| `pones_corne_dongle` + `dongle_screen` (YADS) | `seeeduino_xiao_ble` | 423128 B / 788 KiB (52.44%) | 212548 B (81.08%) | 846336 |

The display-heavy dongle leaves **49596 B** outside linked RAM allocations;
runtime stack/heap headroom has not been measured on hardware. Both
`settings_reset` images also built: nice!nano 46188 B flash / 11552 B RAM /
92672-byte UF2; XIAO 49512 B flash / 11640 B RAM / 99328-byte UF2.
Firmware is under the ignored `build/` directories shown above.

Validation reused an external west/toolchain workspace, passing this checkout's
`config` via `ZMK_CONFIG` and the vendored YADS module via `ZMK_EXTRA_MODULES`.
The Cirque module is discovered through `west update`.
Compiled devicetrees were checked for five scanned columns/four rows per half,
36 transform entries, 36 physical keys, 36 bindings on each of six layers,
the dongle's empty mock scanner and processor chains, right-only Cirque,
dongle-only listener, and split ID 0 on all three. Generated Kconfig confirms
the two peripheral roles, central role, two split peers, seven connection/bond
slots, dongle display/smooth scrolling, and right-side Pinnacle.

There is no native test suite in this configuration repository. A throwaway
`native_posix_64` runtime included the actual dongle overlay and shared keymap;
**12 assertions passed**, producing ten captured mouse reports. It exercised
the production central split-event handler → input-split proxy → listener:
half-speed XY, retained fractional motion, the right MOUSE thumb's 220 ms
hold/release, two-axis scroll while held, pad button press/release passthrough,
left-side fallback right-click/release, cursor restoration, and NAV cursor
movement. Logical key events replace matrix scanning; injected split events
replace BLE delivery; mouse reports are intercepted at the endpoint boundary.
BLE host-profile switching, KVM acceptance, display appearance, physical
scanning, and the Cirque driver's communication with silicon are **not**
simulated or certified by this smoke. No smoke scaffolding is committed.

Non-fatal upstream warnings remain: deprecated `config/boards` discovery and
`NRF_STORE_REBOOT_TYPE_GPREGRET`, peripheral USB defaults, and XIAO devicetree
duplicate-address/deprecated-label warnings, and an `-Wint-conversion` warning
from ZMK's own `central_bas_proxy.c`. Stock settings-reset emits empty-keymap
array warnings.
All five firmware builds complete; there is no module/API build blocker.

## Ported vs. deferred

**Ported:** all six layers in 36-key order, eight home-row mods,
five thumb layer-taps plus a Delete/SYS layer-tap, F+J Caps Word, eight symbol behaviors
(with adaptations above), seven Herdr macros, two KVM macros, media/navigation,
right-peripheral physical pointing, fallback mouse movement/scroll/buttons,
Caps Lock, Insert, and bootloader entry (extended: per-device bootloader and
reset on the ZMK-only SYS layer).

**Deferred / intentional differences:**

- QMK delimiter **hold-to-type pairs** (`{}`, `()`, `<>`, `[]`).
  Stock dances retain single/double taps; type the pair manually.
- QMK one-shot hold symbols, weak-modifier clearing, and exact tap-dance/
  hold-tap/Caps Word timing semantics. Stock ZMK behavior is used instead.
- QMK OLED logos and custom font; the left OLED uses ZMK's stock peripheral
  status screen. The dongle uses YADS's stock status screen.
- QMK console keystroke logger and `DB_TOGG`; that key is blocked.
- Exact QMK kinetic mouse tuning, AVR USB polling/debounce/EEPROM behavior,
  and RGB matrix animations. No in-repo runtime C or driver fork is added.
- Trackpad DR wiring, orientation, and sensitivity come from the previous
  working firmware; real I²C communication with this build still needs hardware
  validation, as does the exact module part number.
- Split pairing, trackpad battery/power behavior, BLE/USB host reports, and
  physical KVM acceptance require the actual keyboard. A local build cannot
  certify these.
- Reviewed but not yet enabled (enable after hardware soak/measurement):
  `CONFIG_ZMK_SLEEP` with an idle timeout on the halves (matrix already has
  `wakeup-source`); hold-tap `require-prior-idle-ms`/`quick-tap-ms` for home-row
  mods; `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN` only if host links drop; LVGL
  pool/font trimming only if the dongle UI misbehaves (dongle RAM is ~81%);
  ZMK Studio on the dongle; moving to ZMK main/Zephyr 4.1.

The six original QMK layers keep their order and bindings. ZMK adds SYS for
device, Bluetooth and output controls, plus two transparent toggle layers for
trackpad speed.

## License

MIT, see [LICENSE](LICENSE). Vendored code under `vendor/zmk-dongle-screen`
keeps its own terms: the module is MIT (`vendor/zmk-dongle-screen/LICENSE`),
and files carrying their own SPDX headers (for example the Apache-2.0 ST7789
display driver) remain under those licenses.

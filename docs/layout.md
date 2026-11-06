# Layout and behavior

The 36-key layout in [`config/pones_corne.keymap`](../config/pones_corne.keymap) has nine layers in this order: QWERTY (0), NUMBER (1), SYMBOL (2), MOVE (3), NAV (4), MOUSE (5), SYS (6), PAD x1 (7), and PAD x6 (8). The two PAD layers are transparent trackpad-speed toggles.

[`config/pones.dtsi`](../config/pones.dtsi) defines stock ZMK behaviors and macros. [`config/pones_corne.conf`](../config/pones_corne.conf) enables pointing and disables underglow. Shield definitions and half-display defaults live in [`config/boards/shields/pones_corne/`](../config/boards/shields/pones_corne/); central settings are in [`config/pones_corne_dongle.conf`](../config/pones_corne_dongle.conf).

## Base layout

Viewed from above:

```text
Q  W  E  R  T       Y  U  I  O  P
A  S  D  F  G       H  J  K  L  ;
Z  X  C  V  B       N  M  ,  .  '
    Esc Tab Space   Enter Bspc Del
```

Home-row keys send their letter on tap and a modifier on hold:

| Key | Hold |
| --- | --- |
| A | Left GUI |
| S | Left Alt |
| D | Left Ctrl |
| F | Left Shift |
| J | Right Shift |
| K | Right Ctrl |
| L | Left Alt (intentional) |
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

Home-row mods and layer-taps are **tap-preferred, 220 ms**. Another key does not force a hold: hold past 220 ms to use the modifier or layer. There are no permissive-hold or bilateral rules. Hold the thumb before using a layer key. Releasing it returns to base even though its position is blocked on the active layer.

**F+J within 30 ms on base**, at zero-based positions **13 + 16**, toggles stock ZMK Caps Word. It uses standard continuation rules and has no idle timeout in ZMK v0.3.0.

## Layers

Unassigned positions on NUMBER, SYMBOL, MOVE, NAV, MOUSE, and SYS are `&none`, **not transparent**: base letters, modifiers, and thumb taps do not leak through. PAD x1 and PAD x6 are fully transparent exceptions.

| Layer | Left hand | Right hand / thumbs |
| --- | --- | --- |
| NUMBER | F1–F4 / F5–F8 / F9–F12 | 7 8 9 `*` / `=` 4 5 6 `+` / 1 2 3 `/`; right thumbs `- . 0` |
| SYMBOL | Home row: Print Screen, Ctrl+C, Ctrl+V, Ctrl+Shift+V | Grave, ampersand/pipe, hash, slash/backslash, caret; `@` and delimiters; tilde, minus/underscore, dollar, percent, question/exclamation |
| MOVE | Herdr commands, Ctrl+Shift+Tab / Ctrl+Tab, KVM previous / next | Previous track, volume down, volume up, next track, play/pause |
| NAV | Home row: GUI, Alt, Ctrl, Shift | Home / Up / End; Page Up / Left / Down / Right / Page Down; Alt+Left / Alt+Right |
| MOUSE | E = up; S/D/F = left/down/right; T/G = scroll up/down; Tab/Space thumbs = right/left click | Y-position = Insert, I = Caps Lock; debug-toggle position is blocked |
| SYS | Q = left bootloader; W = left reset; E / D = toggle trackpad x6 / x1; Z = dongle bootloader (single-key combo); A/S = Bluetooth profile 0/1; C = clear selected host bond; F/G = prefer USB/BLE | O = right reset; P = right bootloader |

The physical trackpad is the primary pointer and moves the cursor on base and every non-MOUSE layer. Hold Backspace/MOUSE past 220 ms to turn pad XY into horizontal/vertical scrolling; release it to restore cursor movement. Native pad taps and thumb mouse buttons remain available. Keyboard movement, scrolling, and buttons provide fallbacks before trackpad wiring/tuning is complete.

Keyboard pointer movement uses linear acceleration over **300 ms** to **1200 units/s**, twice ZMK's stock 600 units/s. This is independent of the physical pad's input-processor tuning; see [trackpad configuration](trackpad.md).

## Symbols: tap dance and hold-tap

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

Delimiter dances use a **180 ms** tapping term. A different key ends the dance early; holding a dance selects the current tap count. The other four symbols are ordinary tap-preferred hold-taps, also 180 ms, not multi-tap dances. Hold outputs remain pressed until release and may auto-repeat on the host; they are not one-shot outputs. Delimiter holds do not type pairs: type both characters manually.

## Macros and KVM

MOVE uses stock ZMK macros:

| Base position | Output |
| --- | --- |
| W / S | Herdr previous / next space: Ctrl+B, then `,` / `.` |
| E / D | Herdr previous / next tab: Ctrl+B, then `p` / `n` |
| Z | Herdr horizontal split: Ctrl+B, then `-` |
| X | Herdr new space: Ctrl+B, then uppercase `N` |
| C | Herdr new tab: Ctrl+B, then `c` |
| T / G | KVM previous / next: Right Ctrl, Right Ctrl, Page Up / Page Down |

The seven Herdr macros use **10 ms presses** and a **10 ms gap** after Ctrl+B. The two KVM macros use **10 ms presses** and **50 ms gaps** between released keys. Modifiers are released between steps. Invoke with other modifiers released: these macros do not clear physically held modifiers. Herdr must recognize those bindings.

KVM sequences target an external KVM's hotkey protocol, **not ZMK Bluetooth profile switching**. Connect the **Prospector dongle's USB** to that KVM. Recognition depends on the KVM and must be checked on hardware.

## SYS: device and host controls

Hold **Delete past 220 ms** to reach SYS. A quick tap still sends Delete; holding Delete does not auto-repeat it.

| Base position | Binding | Effect |
| --- | --- | --- |
| Q / P | `&bootloader` | UF2 bootloader on the half whose key is pressed: left / right |
| W / O | `&sys_reset` | Reboot the left / right half |
| Z (single-key combo) | `&bootloader` | Combos run on the central: UF2 bootloader on the dongle |
| A / S | `&bt BT_SEL 0` / `&bt BT_SEL 1` | Select dongle host profile 0 / 1 |
| C | `&bt BT_CLR` | Clear the selected host profile's bond, not split bonds |
| F / G | `&out OUT_USB` / `&out OUT_BLE` | Prefer USB / Bluetooth host output |

Keyboard-triggered bootloader/reset on a half requires that half to be linked to the dongle because the layer state lives on the central. Otherwise, use its hardware reset button.

Select USB before KVM macros. For a Bluetooth host, select a profile and BLE output. Output preference applies when that transport is ready; ZMK can fall back to an available transport. Profile selection changes neither the KVM's physical port nor the halves' split pairing. Clear a profile only when intending to forget and re-pair its host.

See [build and flash](build-and-flash.md) for the left-then-right pairing procedure and [validation](validation.md) for recorded results and limitations.

## Regenerate the keymap image

The [keymap image](assets/keymap.svg) shows QWERTY, NUMBER, SYMBOL, MOVE, NAV, MOUSE, and SYS. The fully transparent PAD speed layers are excluded. For delimiter dances, the central legend is the single-tap symbol and the upper-right legend is the double-tap symbol; holding emits the single-tap symbol.

From the repository root, use `uvx` and the flash tool's local ZMK workspace to regenerate the image:

```sh
uvx --from keymap-drawer keymap -c docs/assets/keymap-drawer.yaml parse -z config/pones_corne.keymap -o /tmp/pones-keymap.yaml
uvx --from keymap-drawer keymap -c docs/assets/keymap-drawer.yaml draw --dts-layout .zmk/zmk/app/dts/layouts/foostan/corne/5column.dtsi --select-layers QWERTY NUMBER SYMBOL MOVE NAV MOUSE SYS --output docs/assets/keymap.svg /tmp/pones-keymap.yaml
```

Keep [`assets/keymap-drawer.yaml`](assets/keymap-drawer.yaml): it supplies readable macro, mouse, and symbol labels. The parsed `/tmp/pones-keymap.yaml` is temporary and does not need to be committed. If `ZMK_WS` points elsewhere, adjust the `.zmk/` layout path accordingly.

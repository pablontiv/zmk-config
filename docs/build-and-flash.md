# Build and flash

## Build locally with west

No cloud build or CI setup is required. Follow the upstream [ZMK v0.3.0 native toolchain setup](https://v0-3-branch.zmk.dev/docs/development/local-toolchain/setup/native) for host dependencies and the compatible **Zephyr SDK 0.16.8**. On Linux, dependencies include CMake, Ninja, device-tree-compiler, Python with venv support, and SDK host tools. Use a supported Python version for that Zephyr release; Python 3.11 is a conservative choice. The [recorded builds](validation.md) used Python 3.13.5.

From the checkout root, initialize it as a west workspace once. It must not be inside another west workspace. Dependencies and build products stay ignored:

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

Set `ZEPHYR_SDK_INSTALL_DIR` to your actual SDK installation. In subsequent shells, reactivate the venv, set `PONES_CONFIG` and both toolchain variables above, and enter the checkout. `west update` honors the repository's pinned manifest; after pulling changes, run it again if the manifest changed.

Build **all three targets** in separate build directories:

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

The artifacts are `build/pones_corne_{left,right,dongle}/zephyr/zmk.uf2`, in three separate directories. Keep firmware and build products out of Git.

## Flash, reset bonds, and pair

Flashing is scripted by the project skill [`.agents/skills/zmk-flash`](../.agents/skills/zmk-flash/SKILL.md). Run from the repository root with Docker (`zmkfirmware/zmk-build-arm:3.5`) and a west workspace in `ZMK_WS` (default `.zmk/`, gitignored):

```sh
S=.agents/skills/zmk-flash/scripts/zmk-flash
$S all full      # pull, west update (pinned), build, settings_reset + flash all three
$S all halves    # rebuild and flash left then right, keeping bonds
$S mark-good     # after testing: mark the session and tag the known-good flash
```

The flash tool tags each known-good flash. It prints an `ACTION:` line for each physical step. Enter a bootloader using SYS+Z for the dongle, SYS+Q for the left half, SYS+P for the right half, or double-tap the device's reset button. Reach SYS by holding Delete past **220 ms**.

Before writing, the tool identifies each bootloader drive by Board-ID and by comparing its `CURRENT.UF2` with recorded images. It saves that `CURRENT.UF2` under the gitignored `backups/` directory. Use `full` (`--reset`) whenever central/peripheral roles or bonds change: it flashes `settings_reset` before each normal image, erasing bonds and settings, not only firmware.

1. Run the plan and follow its `ACTION:` lines. Keep the halves powered off after flashing until their pairing turn.
2. Plug the dongle into the host or KVM. Power **left only** and let it pair first. Confirm the dongle screen shows its battery before continuing. Then power **right** and let it pair second. This assigns peripheral slot 0 to left and slot 1 to right in the dongle's per-half battery widget.
3. USB host reports now come from the dongle. For Bluetooth, hold **Delete past 220 ms** to reach SYS, press **A or S** for profile **0 or 1**, then **G** for BLE output. Pair **Pones Corne** in the host UI. Forget any stale host bond first to refresh the changed HID descriptor. **SYS+F** selects USB preference for the KVM.
4. Verify all 36 keys, thumb taps/holds, home-row mods, F+J Caps Word, base cursor movement, held MOUSE two-axis pad scrolling, pad taps, thumb buttons, keyboard mouse fallbacks, dongle display, and application/KVM macros. USB on a half is for flashing and power only. Then run `mark-good`.

If pairing fails, follow upstream [ZMK v0.3.0 split connection troubleshooting](https://v0-3-branch.zmk.dev/docs/troubleshooting/connection-issues) and rerun `$S flash full` before pairing **left, then right**.

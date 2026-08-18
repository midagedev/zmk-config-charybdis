# zmk-config-charybdis

ZMK firmware config for a BastardKB Charybdis 4x6 split keyboard:

- **MCU**: nice!nano v2 on both halves (wireless BLE split)
- **Trackball**: PMW3610 on the right half (central)
- **Firmware**: official [ZMK](https://zmk.dev) `v0.3.0` + [badjeff/zmk-pmw3610-driver](https://github.com/badjeff/zmk-pmw3610-driver) (`zmk-0.3` branch)
- **ZMK Studio**: enabled on the central (right) half for live keymap editing

## Building

Push to `main` (or open a PR). GitHub Actions builds via ZMK's reusable workflow and uploads `firmware/*.uf2` artifacts (left, right, settings_reset).

## Flashing

1. Download the artifact `.uf2` files from the Actions run.
2. Double-tap the reset button on the nice!nano to enter UF2 bootloader.
3. Copy the matching `.uf2` onto the mounted drive.

If the halves stop pairing, flash the `settings_reset` firmware to both halves first, then re-flash left/right and re-pair.

## ZMK Studio (UI keymap editing, no reflashing)

The right-half firmware is built with the `studio-rpc-usb-uart` snippet and `CONFIG_ZMK_STUDIO=y`, so you can edit the keymap live:

1. Connect the keyboard to the Mac **over USB** (BLE works only on Linux).
2. Open <https://zmk.studio/> in Chrome/Edge, or install the native macOS app from <https://zmk.studio/download>.
3. Unlock: hold layer 1 (`Space` on the left thumb `&lt 1 SPACE`) and press the `studio_unlock` key (row 3, the old `EP_ON` position).
4. Edit keys/layers in the UI; changes apply immediately. Use "Save" to persist across reboots.

Notes:

- Once you save changes from Studio, edits to `charybdis.keymap` are ignored until you use "Restore Stock Settings" in the Studio UI.
- Two extra empty layers are reserved in the keymap, so new layers can be added entirely from Studio.
- Studio also displays the physical layout and battery level of the connected half.

## Battery

Battery reporting is enabled; macOS shows the charge level under System Settings → Bluetooth (per half, when connected over BLE).

## First-time pointing setup

Enabling `CONFIG_ZMK_POINTING` changes the HID descriptor. Hosts that were paired before must **remove and re-pair** the keyboard, otherwise the trackball will not work over BLE.

# AGENTS.md

ZMK user-config repo for a BastardKB Charybdis 4x6 split (nice!nano v2 on both halves, BLE split) with a PMW3610 trackball on the right half. There is no application source code here; every change is devicetree, Kconfig, or keymap config compiled by the ZMK firmware.

## Build / flash

- There is **no local build**. Push to GitHub; `.github/workflows/build.yml` calls ZMK's reusable workflow (`zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`), which builds the matrix from `build.yaml` (nice_nano_v2 + charybdis_left / charybdis_right / settings_reset) and uploads `firmware/*.uf2` artifacts.
- The repo must stay **public** for the reusable workflow call to work from this account.
- Flash: double-tap reset on the nice!nano to enter UF2 bootloader, copy the matching `.uf2` onto the drive.
- If halves misbehave/pairing breaks: flash `settings_reset` to both halves, then re-flash left/right and re-pair.

## ZMK Studio (owner edits the keymap via UI, not files)

The owner's primary workflow is [ZMK Studio](https://zmk.studio/) — live keymap editing over USB. Consequences for agents:

- `build.yaml` builds the right (central) half with `snippet: studio-rpc-usb-uart` and `cmake-args: -DCONFIG_ZMK_STUDIO=y`. Studio only makes sense on the central; do not add it to the left/settings_reset builds.
- Studio support required converting the shield to a **physical layout**: `charybdis.dtsi` has a `zmk,physical-layout` chosen node (NOT `zmk,matrix-transform`) and a `default_layout` node whose `keys` array (56 entries) must stay **in the same order as the matrix transform `map`**. If you change one, change the other in lockstep.
- The `keys` x/y coordinates are centi-keyunits derived from `config/charybdis.json` (gap between halves at x=6).
- `&studio_unlock` is bound in layer 1 (row 3, col 2 area) — Studio is locked until that key is held/pressed.
- Two `status = "reserved"` empty layers exist so new layers can be created purely in Studio.
- **Gotcha**: once the owner saves keymap changes from Studio, subsequent edits to `charybdis.keymap` are ignored by the firmware until the owner runs "Restore Stock Settings" in the Studio UI. Treat `.keymap` as the stock baseline, not necessarily the live keymap.
- Battery reporting (`CONFIG_ZMK_BATTERY_REPORTING=y`) is enabled; macOS shows per-half battery in Bluetooth settings, Studio shows it for the connected half.

## Firmware stack (version-coupled, handle with care)

`config/west.yml` pins:

- `zmkfirmware/zmk` at `v0.3.0`
- `badjeff/zmk-pmw3610-driver` at branch `zmk-0.3`

**Gotcha**: the PMW3610 module identifiers differ per module branch. On `zmk-0.3` the devicetree compatible is `pixart,pmw3610` and the Kconfig prefix is `CONFIG_PMW3610_*`. On the module's `main`/`zmk-0.4` branches (for ZMK ≥ 0.4) these become `pixart,pmw3610-alt` and `CONFIG_PMW3610_ALT_*`. If you upgrade ZMK, you must bump the module branch **and** update `charybdis_right.overlay` + `charybdis_right.conf` together, or the build fails / driver silently missing.

## Repo layout

| File | Purpose |
| --- | --- |
| `config/west.yml` | West manifest; pins ZMK + PMW3610 module versions |
| `build.yaml` | GitHub Actions build matrix |
| `config/charybdis.keymap` | Keymap (2 layers), ESC combo, scroll-layer input processor |
| `config/charybdis.conf` | Global config (`CONFIG_ZMK_POINTING`, BT TX power) |
| `config/boards/shields/charybdis/` | Shield definition: kscan matrix, matrix transform, SPI/trackball wiring |
| `config/charybdis.json` | Physical layout metadata for the keymap editor (do not hand-edit) |

## Split roles and trackball architecture

- **Right half = central** (`ZMK_SPLIT_BLE_ROLE_CENTRAL`, keyboard name "Charybdis"), and it hosts the trackball. Left half = peripheral.
- The trackball driver node lives under `&spi0` in `charybdis_right.overlay`; it is a 3-wire SPI setup (MOSI and MISO share pin 17), CS on `gpio0 20`, IRQ on `gpio0 6`, SCK on pin 8. Pin mappings come from the physical build; do not "clean them up".
- `charybdis.dtsi` defines `trackball_listener` (`zmk,input-listener`) with `status = "disabled"`, so the keymap can reference `&trackball_listener` in **both** builds; only the right overlay enables it and points it at the sensor. Keep this pattern if you restructure.
- Scrolling: the keymap attaches a child node to `&trackball_listener` that applies `zip_xy_to_scroll_mapper` when layer 1 is active. Axis orientation is fixed in the overlay with `swap-xy; invert-x;` (this replaces the old fork's `CONFIG_PMW3610_ORIENTATION_*` Kconfigs, which no longer exist).

## Keymap conventions

- Layer 0 = Base; layer 1 = nav/function layer (`&lt 1 SPACE` home-thumb hold; `&studio_unlock`, `&bt BT_CLR`, arrows, F-row).
- ESC via combo (positions 17 18), not a dedicated key.
- `&mkp MB1` on a thumb key is the mouse left-click.
- The owner changes the keymap through ZMK Studio at runtime; file edits to `.keymap` are only the stock baseline. Expect whole-file rewrites when the keymap editor is used; review diffs structurally (per-key positions), not by line.

## Tuning knobs

- Cursor speed: `cpi = <600>` in `charybdis_right.overlay` (PMW3610 range 200–1200).
- Hold-tap feel: the `&lt { ... }` block at the top of the keymap (tapping-term 200 ms, balanced, quick-tap 150 ms).
- Scrolling speed: add `&zip_scroll_scaler <mul> <div>` to the scroller child node's `input-processors`.

## Known troubleshooting

- `Incorrect product id 0xFF (expecting 0x3E)!` on nice_nano v2 at boot: add `CONFIG_PMW3610_INIT_POWER_UP_EXTRA_DELAY_MS=1000` to `charybdis_right.conf` (sensor power rail needs time after ext power enables).
- After first enabling `CONFIG_ZMK_POINTING`, previously paired hosts must remove and re-pair the keyboard, or the trackball does nothing over BLE (changed HID descriptor).
- Legacy BT tuning from the old firmware (LLCP legacy, conn interval/buffer counts) was intentionally dropped in the migration to v0.3.0. Re-introduce only if BLE stability regresses.

## History

Migrated 2026-08-18 from a 2023 custom fork (`Amos698/zmk` `feature-test` with in-tree mouse code, predecessor repo `zmk-for-charybdis`). Old fork Kconfigs (`CONFIG_ZMK_MOUSE*`, `CONFIG_PMW3610_ORIENTATION_*`, `CONFIG_ZMK_PD_*`, `CONFIG_ZMK_TRACKBALL_POLL_DURATION`) do not exist in this stack.

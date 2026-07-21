# Nice!View Gem Setup — 2026-06-24

## Current state
- **Branch**: `master` on ZMK main (Zephyr 4.1 / LVGL 9)
- **Display**: nice-view-gem with custom peripheral battery support (forked)
- **Mode**: Battery bars enabled (`CONFIG_NICE_VIEW_GEM_BATTERY_BARS=y`)

## Config files changed
- `config/west.yml` — uses `oresk/nice-view-gem` fork, branch `peripheral-battery`
- `config/splitkb_aurora_lily58.conf` — added:
  - `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`
  - `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y`
  - `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y`
  - `CONFIG_NICE_VIEW_GEM_BATTERY_BARS=y`
- `config/splitkb_aurora_lily58.keymap` — added `display-name` labels to all layers
- `config/splitkb_aurora_lily58_left.overlay` — disables `&oled` node (pin conflict fix)
- `build.yaml` — left half shield: `splitkb_aurora_lily58_left nice_view_adapter nice_view_gem`

## Fork location
- **Repo**: https://github.com/oresk/nice-view-gem
- **Branch**: `peripheral-battery` (synced from upstream `M165437/nice-view-gem` main)
- **Changes**:
  - `util.h` — added `peripheral_battery` + `peripheral_connected` to central `status_state`
  - `screen.c` — added subscriptions to `zmk_peripheral_battery_state_changed` and `zmk_split_peripheral_status_changed`
  - `battery.c` — dual battery bar gauges (L/R, drain opposite ways), bars/text toggle via `CONFIG_NICE_VIEW_GEM_BATTERY_BARS`, bolt icon 5px right + 3px down from original
  - `output.c` — removed "SIG" label, kept BT/USB icons at original top-right positions (45-49)
  - `Kconfig.defconfig` — added `CONFIG_NICE_VIEW_GEM_BATTERY_BARS` option

## Display layout (bar mode)
```
                              [BT/USB]
  ┌──────┐     ⚡     ┌──────┐    ← bars at edges
  │████░░│           │░░████│    ← L dark anchored left, R dark anchored right
  └──────┘           └──────┘
```
- Left bar: dark fill anchored **left**, empties from right as battery drains
- Right bar: dark fill anchored **right**, empties from left
- Max fill capped at 99%
- Lightning bolt between bars when charging

## Pending
- Flash the latest firmware to left nice!nano (build `1b49b5c` or later)
- Double-tap reset on left nice!nano → `NICENANO` drive appears → copy UF2

## To continue
1. Wait for latest GitHub Actions build to finish
2. Download artifact → extract UF2
3. Flash left half
4. Test: layer names on bottom, profile dots switch with BT profiles, both battery bars

# BT flapping + USB freeze — display removal + sleep (2026-07-21)

## Problem
- Over Bluetooth: host connection flaps (connect/disconnect) several times per
  second.
- Over USB: keyboard works for a while, then freezes; needs the reset button.
- Both transports misbehave → points at firmware instability, not a host/pairing
  or cable issue (USB-C cable/source confirmed solid).

## Likely root cause
- ZMK `main` (unreleased, ~11 months ahead of stable v0.3.0) + custom LVGL-9
  display widget (`oresk/nice-view-gem` branch `peripheral-battery`).
- History (`session-notes.md`): an earlier ZMK main + LVGL-9 display fork
  "crashed (no USB device, blank display)". The same fragility now manifests as
  repeated faults instead of a total crash.
- Display widget load on the central: subscribes to ~8 event types, redraws
  64x64 canvases + software-rotates on each; WPM gauge redraws per keystroke;
  `output.c` redraws twice on every USB/BLE-profile/endpoint change.

## Changes made (isolation + power)
- `config/splitkb_aurora_lily58.conf`
  - Commented out `CONFIG_ZMK_DISPLAY`, `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM`,
    `CONFIG_NICE_VIEW_GEM_BATTERY_BARS`, split central battery fetch/proxy
    (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING/PROXY`).
  - `CONFIG_ZMK_SLEEP=n` → `CONFIG_ZMK_SLEEP=y` (deep sleep on both halves).
  - Added `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=300000` (5 min, lower than 15-min
    default so sleep can be verified during testing).
- `build.yaml`
  - Left shield: `splitkb_aurora_lily58_left nice_view_adapter nice_view_gem`
    → `splitkb_aurora_lily58_left` (no display shield).
- `config/splitkb_aurora_lily58_left.overlay` — left unchanged (disabling
  `&oled` is harmless with no display/I2C enabled).
- `config/west.yml` — left unchanged; still tracks ZMK `main` (build fetches
  latest main via GitHub Actions). Per user: try latest main, no pin.

## What this tests
- Removing all LVGL/draw code from the central eliminates the display CPU load
  as a fault source. If the flapping/freeze stops, the display widget (or its
  interaction with ZMK main) was the cause.
- Enabling sleep on both halves is the biggest battery lever, especially for the
  peripheral (previously never slept with `CONFIG_ZMK_SLEEP=n`).

## To verify
1. Push, let GitHub Actions build, download artifact, flash both halves.
2. Test BT: leave it connected idle and typing for a while — connection should
   stay stable, no flapping.
3. Test USB: leave connected, confirm no freeze / no reset needed.
4. Test sleep: leave idle ~5 min, confirm it sleeps and wakes on keypress on both
   halves; confirm battery lasts longer.

## If stable
- Re-enable a display: first try the **built-in** OLED status screen
  (`CONFIG_ZMK_DISPLAY=y` + built-in screen, enable SSD1306/I2C, remove the
  `&oled`-disable overlay) rather than the custom nice-view-gem widget, to keep
  the draw load low. Only re-add nice-view-gem if the built-in screen is stable.
- Consider pinning ZMK to stable `v0.3.0` (west.yml `revision: v0.3.0` and
  `build.yml` workflow `@v0.3.0`) instead of `main` for long-term stability.

## If still flapping/freezing with display removed
- Then the display is not the cause; next steps: enable USB logging
  (`-DCONFIG_ZMK_USB_LOGGING=y`) build to capture the fault, and/or pin ZMK to
  `v0.3.0` to rule out a `main` regression.
---

## Resolution (same day, 2026-07-21)

### Root cause confirmed
The peripheral-battery fork of nice-view-gem (oresk/peripheral-battery) was the
cause of both BT flapping and USB freezing. The fork added:
- Extra BLE event subscriptions (peripheral battery + split status) on the
  central, triggering screen redraws on every peripheral battery/split event
- Dual battery bar rendering (twice the draw calls per refresh)
- BLE polling traffic (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING`,
  `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY`)

These combined with LVGL's software-rotation `rotate_canvas()` overloaded the
nice!nano's CPU on every redraw, destabilising both transports.

### Final state
- **west.yml**: upstream `M165437/nice-view-gem` `main` (LVGL9-compatible)
- **build.yaml**: left shield includes `nice_view_adapter nice_view_gem`
- **conf**: `CONFIG_ZMK_DISPLAY=y`, `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`,
  deep sleep enabled (`CONFIG_ZMK_SLEEP=y`, 5 min idle timeout)
- **BT**: stable, no flapping
- **USB**: stable, no freezes
- **Display**: working (upstream gem — single battery, layer name, output icons)
- **Split**: halves paired and communicating
- **Sleep**: to be verified after long idle

### If dual battery bars are wanted later
Re-add peripheral battery support on a stable foundation:
1. Pin ZMK to `v0.3.0` first (west.yml + build.yml `@v0.3.0`)
2. Then re-introduce battery bars with smaller subscriptions, or use a
   polling interval instead of per-event redraws

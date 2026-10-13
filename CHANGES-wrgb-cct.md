# WRGB color order + persistent white-balance (CCT) patch

This branch (`feature/wrgb-cct-on-latest-upstream`) rebases dev-ath0m's
custom changes onto current upstream WLED (`wled-dev/WLED@main`).

## 1. WRGB color order support

Adds `COL_ORDER_WRGB` (value 6) for SK6812-style RGBW strips that transmit
the White byte first on the wire (W, R, G, B), instead of one of the six
existing RGB/RGBW orderings.

- `wled00/const.h`: added `COL_ORDER_WRGB` constant, bumped `COL_ORDER_MAX` to 6.
- `wled00/bus_wrapper.h`: `PolyBus::setPixelColor()` / `getPixelColor()` map
  W/R/G/B to/from the wire order when color order 6 is selected.
- `wled00/data/settings_leds.htm`: added a "WRGB" option to the color-order
  dropdowns in the LED settings UI.

Select color order "WRGB" on a 4-channel (RGBW) bus to use it.

## 2. Persistent white-balance (CCT) via cfg.json

WLED applies a Kelvin-based white-balance correction per segment via the
`cct` (0-255) property when a bus has `correctWB` enabled. Normally this
value is only restorable via presets, but preset-save is broken on this
fork's boot flow, so the value was always reset to default (127, neutral)
on every reboot.

This patch persists a default CCT value directly in `cfg.json` and applies
it to all segments at boot, independent of presets:

- `wled00/wled.h`: new global `defaultCCT` (`int16_t`, default `-1` = unset,
  meaning "use compiled default").
- `wled00/cfg.cpp`: reads/writes `defaultCCT` under the `def.cct` key in
  `cfg.json` (`deserializeConfig()` / `serializeConfig()`).
- `wled00/wled.cpp`: in `beginStrip()`, if `defaultCCT >= 0`, applies it to
  every segment via `setCCT()` right after segments are created.

To set it, `POST` to `/json/state` as usual to find the desired `cct` value,
then persist it with:

```
curl -s http://<device-ip>/json/cfg -d '{"def":{"cct":<value>}}' -H "Content-Type: application/json"
```

The value now survives reboots without relying on presets.

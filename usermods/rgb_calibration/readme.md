# RGB Calibration

This ESP32-only usermod applies independent RGB output gains to one existing WLED digital LED output. It is intended for manual visual matching of two LED strips, not automatic color measurement.

## Installation

Add `rgb_calibration` to `custom_usermods` in a local PlatformIO environment, then build normally. It requires the RGB calibration core support included with this change and does not allocate, configure, or own a GPIO.

This implementation follows WLED's [EXAMPLE registration/configuration pattern](https://github.com/wled/WLED/tree/v16.0.1/usermods/EXAMPLE). The core extension is in `BusDigital::setPixelColor()`, not an effect or overlay. Use a firewall-isolated LAN as with normal WLED installations.

For QuinLED Dig-Uno v3.1r2 or v3.5r3 with the classic QuinLED-ESP32 Ethernet module, the local ignored `platformio_override.ini` can contain:

```ini
[env:quinled_dig_uno_v31r2]
extends = env:esp32_eth
custom_usermods =
  ${env:esp32_eth.custom_usermods}
  rgb_calibration
build_flags =
  ${common.build_flags}
  ${esp32.build_flags}
  -D WLED_RELEASE_NAME=\"ESP32_Ethernet_RGB_Calibration\"
  -D RLYPIN=-1
  -D BTNPIN=-1
  -D WLED_USE_ETHERNET
  -D SR_DMTYPE=-1
  -D AUDIOPIN=-1
  -D I2S_SDPIN=-1
  -D I2S_WSPIN=-1
  -D I2S_CKPIN=-1
  -D MCLK_PIN=-1
  -D DATA_PINS=3
  -D WLED_ETH_DEFAULT=4
  -D WLED_DISABLE_ESPNOW
  -D WLED_DISABLE_ADALIGHT
  -D ARDUINO_USB_CDC_ON_BOOT=0
```

Build with `pio run -e quinled_dig_uno_v31r2` after `npm ci` and `npm run build`. The flags replace the base environment's GPIO4 default while retaining its Ethernet/audio safeguards. Saved device settings override these defaults. GPIO3 is UART RX, so this build deliberately disables serial-input protocols; ESP-NOW is also disabled for the Ethernet clock configuration. Verify board routing and strip timing before deployment. Do not change an output's protocol merely to enable calibration.

The environment retains its original `v31r2` name; the same LED2/GPIO3 and Ethernet type 4 configuration applies to v3.5r3. The [official revision history](https://quinled.info/quinled-dig-uno-pre-assembled-revision-history/) lists power-circuit and EMI/EMC changes for v3.5, rather than a new LED pinout. This LED2-focused build does not preconfigure LED1, a button, or a relay; it is not a full replacement for all defaults in QuinLED's own firmware builds.

## Settings

The Usermod Settings page stores these values under `RGB Calibration`:

| Setting | Default | Meaning |
| --- | --- | --- |
| `enabled` | `false` | Enables correction while retaining saved gains when disabled. |
| `outputGpio` | `3` | Existing WLED LED-data GPIO. Use `-1` to select no output. |
| `redGain`, `greenGain`, `blueGain` | `100` | Per-channel percentage from `0` to `200`. |

Only native `WS281x RGB` and `400kHz RGB` outputs are calibrated. RGBW/CCT, PWM, two-pin, network-output, HUB75, status-pixel, and other output types are unchanged. The Info page reports whether the selection is disabled, active, missing, unsupported, or invalid/placeholder.

## Signal Scope

Correction happens in the selected physical output path after WLED's color processing and before bus brightness and current limiting. It affects native effects and incoming Art-Net, E1.31, and DDP data on the next normal output frame. It does not modify composite/effect buffers, and it does not calibrate outgoing network LED data.

Incoming UDP realtime is also covered. Skipping realtime gamma does not skip calibration, but it can change the visual match. Live LEDs and other logical-frame exports remain uncalibrated. A configuration change uses one complete settings tuple per pixel, but may take effect partway through a frame; frame-atomic updates are not promised. A valid GPIO selection stays dormant until a supported output exists on it, and automatically follows that output after bus reconstruction.

Each channel is independently rounded and clipped to 255. A gain above 100% can clip before master brightness is applied, so lowering brightness cannot recover highlights. Prefer reducing a stronger channel to match another strip. Disabled and 100/100/100 settings are exact bypasses. During realtime input, changes take effect on the next received frame/show; an idle realtime stream is not forcibly shown.

## Matching Procedure

1. Use matching RGB test colors, gamma settings, color-temperature/white-balance settings, power conditions, and viewing conditions on both strips. Verify RGB color order first.
2. Start with all gains at 100% and compare a mid-to-high neutral gray. Reduce the channels that appear excessive; adjust overall brightness separately.
3. Compare neutral values such as 16, 32, 64, 128, 192, and 255, then primaries and secondaries at multiple master-brightness levels.
4. Confirm persistence across reboot, disable-to-stock behavior, selected-output isolation, realtime updates, bus reconstruction, and whether ABL is limiting output.

Independent gains cannot guarantee a match at every color or brightness. Intensity-dependent tint can require channel curves, while differing LED spectra can require a color matrix; neither is implemented here. Bus transport readback includes calibration and must not be fed back into the same output for correction.

Do not disable electrical protections to compare brightness. ABL must account for the calibrated values, but its model is only an estimate; verify the 24V strip's addressable group count, current draw, wiring, and fusing independently.

## Host Checks

Run `node usermods/rgb_calibration/tests/run_host_checks.js` from the repository root. It needs a C++11 host compiler, default `g++`; `CXX` can name another GCC-compatible compiler executable. On Windows, put the compiler's runtime DLL directory on `PATH` as well. The runner builds in a temporary directory and fails clearly if the compiler or checks fail. It is deliberately separate from `npm test`.

Checks exercise production arithmetic and JSON helpers, not the hardware driver. Firmware builds, real LED timing, save/reboot persistence, and visual matching are distinct validation steps. Settings are normal device configuration, not preset state. An old configuration lacking the root `um` object keeps defaults in memory until the next normal configuration save.

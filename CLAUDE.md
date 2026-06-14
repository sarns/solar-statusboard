# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **ESPHome** firmware configuration for a solar power and weather monitoring dashboard. It runs on an ESP32-S3 microcontroller driving a T547 4.7" e-ink display. All data is pulled in real time from **Home Assistant** sensors.

The entire firmware is defined in a single file: `solar-statusboard.yaml`.

## ESPHome CLI Commands

ESPHome must be installed separately (`pip install esphome`). There is no build system in the repo itself.

```bash
esphome validate solar-statusboard.yaml   # Check YAML syntax and config validity
esphome compile solar-statusboard.yaml    # Compile firmware only
esphome run solar-statusboard.yaml        # Compile and flash via USB
esphome upload solar-statusboard.yaml     # OTA flash to a running device
esphome logs solar-statusboard.yaml       # Stream device serial logs
esphome clean solar-statusboard.yaml      # Remove build artifacts
```

Secrets (`wifi_ssid`, `wifi_password`, encryption key, etc.) must be defined in a `secrets.yaml` file in the same directory — this file is not committed to the repo.

## Architecture

### Data Flow

```
Home Assistant sensors → ESPHome homeassistant platform → Global sensor state → Display lambda
```

The device connects to Home Assistant via the native API. All 22 sensors (weather, solar, battery, temperatures) are declared as `sensor` or `text_sensor` components with `platform: homeassistant`, binding to HA entity IDs.

### Configuration Layers (in order within the YAML)

1. **ESPHome core** — device name, board (esp32-s3-devkitc-1), API encryption, OTA, WiFi
2. **External component** — t547 e-ink display driver pulled from `github://jesuli87/esphomeStuff`
3. **Fonts** — Roboto (14–64pt) and Material Design Icons (48–55pt) via Google Fonts URLs
4. **Sensors** — 4 weather forecast sensors, 4 weather condition text sensors, 8 solar/power sensors, 7 room temperature sensors
5. **Globals** — `display_page`, `page_refresh_default_s` (60s), `initial_sensor_sync_pending`
6. **Script** (`manage_run_and_sleep`) — main loop: wait for HA + time sync → init sensors (5s) → refresh display → sleep → repeat
7. **Display** — single `t547` display component with one full-screen rendering lambda

### Display Rendering

The display lambda is pure C++ embedded in YAML. Key conventions:
- **Color model is inverted**: `Color(0, 0, 0)` = white, `Color(255, 255, 255)` = black (e-ink requirement)
- Color constants are defined at the top of the lambda (`COLOR_ON`, `COLOR_OFF`, etc.)
- Layout uses absolute pixel coordinates targeting a ~960×540 resolution
- Greyscale is achieved via a checkerboard dithering helper lambda
- Weather icons and battery icons use Material Design Icons codepoints

### Hardware Dependencies

- **SPIRAM** is required (OCT mode) — configured via `sdkconfig_options` in the YAML
- **Legacy RMT driver** must be enabled (`CONFIG_RMT_LEGACY_IDF: "y"`) for display communication
- The t547 external component must be fetched at compile time from GitHub

## Key Conventions

- All sensitive values use `!secret <key>` — never hardcode credentials
- Sensor entity IDs follow HA naming: `sensor.<entity_name>` — update these if the HA instance changes
- The 60-second refresh interval (`page_refresh_default_s`) balances e-ink longevity vs. data freshness
- When adding new sensors, declare them in the `sensor`/`text_sensor` section before referencing them in the display lambda
- The display lambda runs synchronously; heavy computation should be avoided or pre-calculated in sensor `on_value` filters

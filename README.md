# big-PrintSphere (ESPHome + LVGL, Round AMOLED 1.75")

An ESP32-S3 round display for Bambu Lab / Home Assistant print status.

Features:
- large progress ring
- 3-page LVGL UI:
  - Page 1: print status, layer, remaining time, temperatures
  - Page 2: round print preview image
  - Page 3: HA camera view via `camera_proxy`
- status-based colors and animations
- touch + brightness gesture + power management (AXP2101)

---
Requires Home Assistant, Bambu integration entities, and a connected ESPHome device.

## Hardware

- **Waveshare ESP32-S3 Touch AMOLED 1.75" (466x466)**  
  ESPHome: `display: platform: mipi_spi` (quad SPI bus)
- **Touch**: CST9217
- **PMU**: AXP2101

## Dependencies

- ESPHome (current release recommended)
- LVGL via ESPHome `lvgl:`
- Home Assistant with Bambu entities
- external components used by this config:
  - `https://github.com/shelson/esphome-cst9217`
  - `https://github.com/cptkirki/AXP2101_WS-AMOLED-1.75`

## Configuration (important substitutions/secrets)

```yaml
substitutions:
  homeassistant_url: http://homeassistant.local:8123
  bambulab_printer: p1s_01p00a546874654
  bambulab_icon: images/bambuicon.png
  ha_cam_entity_id: bigsphere_cam   # suffix only, NOT "camera.bigsphere_cam"
```

`secrets.yaml` must include:

```yaml
ha_auth_header: "Bearer <LONG_LIVED_ACCESS_TOKEN>"
```

Notes:
- `homeassistant_url` must be reachable from the ESP device.
- This YAML currently uses `http_request.verify_ssl: false` for convenience.

## Expected Home Assistant entities

Required:
- `sensor.<printer>_current_stage`
- `sensor.<printer>_print_status`
- `sensor.<printer>_print_progress`

Optional (extra info):
- `sensor.<printer>_nozzle_temperature`
- `sensor.<printer>_bed_temperature`
- `sensor.<printer>_remaining_time`
- `sensor.<printer>_current_layer`
- `sensor.<printer>_total_layer_count`

Optional (error flags):
- `binary_sensor.<printer>_hms_errors`
- `binary_sensor.<printer>_druckfehler` (need to be translated to other language if needed)

**Image entity (for Page 2, round preview)**
- `image.<printer>_titelbild` (its `entity_picture` is used) 

> Without `image.<printer>_titelbild`, Page 2 shows “Preview disabled”.

Camera page (Page 3):
- `camera.<ha_cam_entity_id>`  
  Example with default substitution: `camera.bigsphere_cam`

## Home Assistant Proxy Cam (important)

Page 3 loads frames from:

```text
/api/camera_proxy/camera.<ha_cam_entity_id>?width=400&height=225&time=<timestamp>
```

The ESP sends the `Authorization` header from `!secret ha_auth_header`.

Important details:
- `ha_cam_entity_id` is only the suffix (example: `bigsphere_cam`), because the URL already prepends `camera.`
- token must be a valid HA Long-Lived Access Token (`Bearer ...`)
- if you use HTTPS with self-signed certs, either keep `verify_ssl: false` or provide proper cert handling

Quick test from another machine in your LAN:

```bash
curl -H "Authorization: Bearer <TOKEN>" \
  "http://<HA_HOST>:8123/api/camera_proxy/camera.bigsphere_cam?width=400&height=225" \
  --output frame.jpg
```

If this returns an image, ESP page 3 should work too.

## UI logic (short)

- Filament loading/unloading: yellow animated ring (priority)
- Error: pulsing red ring
- Done (strict): only when
  - `print_status` indicates finish
  - stage is idle
  - progress is 100
- Idle (not done): gray
- Printing/prepare stages: stage-dependent ring colors and blending

## Setup

1. Configure substitutions and secrets.
2. Flash `big-printsphere.yaml` to the ESP32-S3 board.
3. Verify HA entities exist, especially:
   - `image.<printer>_titelbild` (Page 2)
   - `camera.<ha_cam_entity_id>` + valid `ha_auth_header` (Page 3)
4. Swipe horizontally to switch pages.

## Troubleshooting

- Page 2 says no/old preview:
  - check `image.<printer>_titelbild`
  - check if HA provides a valid `entity_picture`
- Page 3 stays blank:
  - verify `camera.<ha_cam_entity_id>` exists
  - verify `ha_auth_header` format is exactly `Bearer <token>`
  - test camera proxy endpoint with `curl`
- 401/403 on camera proxy:
  - token missing/invalid/expired
- 404 on camera proxy:
  - wrong `ha_cam_entity_id` (or included `camera.` twice)

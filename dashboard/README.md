# Dashboard cards

Two ready-to-use Lovelace cards for the Luke Roberts gateway:

- [`lamp-card.yaml`](lamp-card.yaml) – everyday controls (scenes, brightness,
  color temperature). Core HA cards plus optionally the
  `custom:mushroom-select-card` (HACS) for a nicer scene picker.
- [`diagnostics-card.yaml`](diagnostics-card.yaml) – admin-only card with
  Ping / Restart / Refresh scenes / last API response / device info. Gated
  by a `visibility:` block on HA user id + mains power state.

## Entities referenced

| Role                    | Entity ID                                                            |
| ----------------------- | -------------------------------------------------------------------- |
| Mains power             | `light.your_lamp_power_switch` *(placeholder – replace with your switch)* |
| BLE reachability        | `binary_sensor.esp32_s3_luke_roberts_gateway_lamp_ble_presence`      |
| Scene dropdown (named)  | `input_select.lr_scene` *(optional, from the home-assistant/ package)* |
| Scene dropdown (by id)  | `select.esp32_s3_luke_roberts_gateway_scene`                         |
| Lamp Off                | `button.esp32_s3_luke_roberts_gateway_lamp_off`                      |
| Default Scene           | `button.esp32_s3_luke_roberts_gateway_default_scene`                 |
| Scene 1 – 15            | `button.esp32_s3_luke_roberts_gateway_scene_1` … `_scene_15`         |
| Next brighter / dimmer  | `..._next_brighter_scene` / `..._next_dimmer_scene`                  |
| Brightness ±20 % / 50 % | `..._brighter_20` / `..._dimmer_20` / `..._brightness_50`            |
| 3000 K preset           | `button.esp32_s3_luke_roberts_gateway_color_temperature_3000k`       |
| Immediate DL 50% 3000K  | `button.esp32_s3_luke_roberts_gateway_immediate_downlight_50_3000k`  |
| Ping                    | `button.esp32_s3_luke_roberts_gateway_ping`                          |
| Restart gateway         | `button.esp32_s3_luke_roberts_gateway_restart`                       |
| Refresh scene list      | `button.esp32_s3_luke_roberts_gateway_refresh_scenes`                |
| API response            | `sensor.esp32_s3_luke_roberts_gateway_luke_roberts_api_response`     |
| Firmware / Model / Serial | `sensor..._firmware_revision` / `..._model_number` / `..._serial_number` |

> Adjust the entity ids if your ESPHome device has a different `friendly_name`
> (entity ids are derived from `friendly_name` + entity `name`).

## Main card (`lamp-card.yaml`)

Three states are selected automatically via `conditional` cards:

1. **Mains off** – only the status glance row is shown (BLE will be `off`).
2. **Mains on, BLE not reachable yet** – a short "waiting" markdown notice.
3. **Mains on, BLE reachable** – full control panel:
   - Off / Default quick buttons
   - Pretty-named scene picker via `custom:mushroom-select-card` bound to
     `input_select.lr_scene` (requires the HA-side package in
     [`../home-assistant/`](../home-assistant/))
   - Dimmer / Brighter scene navigation
   - Brightness −20 % / 50 % / +20 %
   - 3000 K preset and Immediate DL 50 % 3000 K

If you don't have Mushroom installed via HACS, replace the
`custom:mushroom-select-card` block with a plain `entities` card listing
`input_select.lr_scene`. Everything else uses only core HA cards.

> Native Lovelace button cards don't support template labels, so the scene
> buttons would keep fixed `Scene N` names. That's why the primary picker
> here is the HA-side `input_select` helper, which carries the discovered
> names ("1: Welcome!", "7: Bright", …).

## Admin card (`diagnostics-card.yaml`)

A second card, kept separate so it can be hidden from everyday views. It
exposes the ESPHome-side diagnostic entities (Ping, Restart, Refresh scenes,
last API response, firmware / model / serial) and is gated by a `visibility:`
block with two conditions:

- **User condition** – only render for a specific HA user id.
  Replace `REPLACE_WITH_YOUR_HA_USER_ID` with your own user id (Settings
  → People → pick a user → "User ID" is a 32-char hex string).
- **State condition** – only render while the mains switch is on.
  Replace `light.your_lamp_power_switch` with your actual switch entity.

Drop either condition if you don't need it.

## Install

1. Open Home Assistant → **Dashboards** → *edit* the target dashboard.
2. **Raw configuration editor** → paste the contents of
   [`lamp-card.yaml`](lamp-card.yaml) into a view's `cards:` list.
3. Paste [`diagnostics-card.yaml`](diagnostics-card.yaml) into the same view
   (or a separate admin view) if you want the diagnostics panel too.
4. Save.

Alternatively, keep the snippets in a dashboard package under
`dashboards/packages/` if you use YAML-mode dashboards.
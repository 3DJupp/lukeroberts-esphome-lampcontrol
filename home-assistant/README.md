# Home Assistant side-configuration

ESPHome's `select` entity has a *static* options list — HA only pulls it once
when the gateway connects, so a dynamic "fill the dropdown with the real
scene names I just discovered" can't be done cleanly from ESPHome itself.

The neat workaround is to keep the pretty-named dropdown on the HA side
instead, as an `input_select` helper that's refreshed from the gateway's
scene-name sensors via an automation. You still get a single dropdown with
proper labels like `"1: Welcome!"`, `"2: Highlights"`, …, and every selection
translates one-to-one into the matching button press on the gateway.

## Install

1. Make sure packages are enabled in your `configuration.yaml`:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. Copy [`packages/lukeroberts.yaml`](packages/lukeroberts.yaml)
   into `<HA_config>/packages/`.

3. Restart Home Assistant.

An `input_select.lr_scene` helper will appear, starting with just `Default`.
As soon as the gateway has walked the scene list (on connect or after a
**Refresh Scenes** press), the dropdown repopulates with
`Default, 0: Off, 1: Welcome!, 2: Highlights, …` — only entries with a
discovered name are included.

## How it works

Two automations:

- **LR Scene – refresh options** rebuilds the option list whenever the
  gateway republishes `sensor..._scenes_overview`. The list uses the
  format `"N: Name"` so that even duplicate names (`"10: Bright"` vs.
  `"7: Bright"`) stay uniquely addressable. `input_select.set_options` is
  only called when the new list differs from the current one, to avoid
  nudging the selection state on unrelated updates.
- **LR Scene – apply selection** runs on every user-initiated change of
  the dropdown (guarded on `context.user_id` so programmatic updates don't
  re-fire). It parses the leading id and calls `button.press` on
  `button.<prefix>_default_scene`, `_lamp_off`, or `_scene_<id>` as
  appropriate.

## Adjusting entity names

The package assumes the ESPHome device's friendly name is
`ESP32-S3 Luke Roberts Gateway`, yielding entity prefixes like
`esp32_s3_luke_roberts_gateway_*`. If yours differs, find-and-replace that
prefix inside `packages/lukeroberts.yaml`.

## Caveats

- Selecting a scene via the dropdown doesn't update the ESPHome-side
  `select.*_scene` entity (they're independent). If you want a single
  source of truth in the UI, just hide one of them.
- If a scene is renamed in the Luke Roberts app while HA is running, click
  **Refresh Scenes** on the gateway — the dropdown will update within a
  couple of seconds.
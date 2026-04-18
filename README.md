# Luke Roberts Lamp – ESPHome Gateway

ESPHome configuration that turns an **ESP32** into a Bluetooth-LE gateway for
**Luke Roberts** smart lamps (Model F / Luvo). Exposes the lamp as a set of
Home Assistant buttons, binary sensors and text sensors — no mobile app
required after initial setup.

Built against the official **[Lamp Control API v1.3](/docs/Lamp%20Control%20API%20v1.3.pdf)**
(Luke Roberts, 2019-07-09).

## Features

- Scene selection, on/off, default scene
- Brightness adjustment (absolute percent, relative multiplier, next brighter / dimmer)
- Color temperature (2700 K – 4000 K)
- Immediate Light command (transient brightness + kelvin)
- Live protocol response decoding (OK / Invalid Id / Bad Command / Forbidden …)
- Device Information read-out (manufacturer, model, serial, FW/HW/SW revisions)
- Ping-based keepalive so successive commands fire in < 100 ms

## Hardware

- Any ESP32 board with BLE. Tested on an **ESP32-S3-WROOM-1**
  (`esp32-s3-devkitc-1`). Change the `esp32.board` value in
  [`luke-roberts-lamp.yaml`](luke-roberts-lamp.yaml) for other modules.
- A Luke Roberts smart lamp within Bluetooth range (typically < 10 m line of sight).

## Quickstart

1. **Clone the repo** into your ESPHome config directory, or drop
   `luke-roberts-lamp.yaml` into it directly.

2. **Find your lamp's MAC address.** Easiest way: power on the lamp, open the
   Luke Roberts app and connect to it — the MAC is shown in the lamp's
   Bluetooth details on most phones. Alternatively, temporarily add
   `esp32_ble_tracker:` to any ESPHome config and grep the log for
   `LRBTLAMP` / `Luvo` / the lamp's advertised name.

3. **Configure secrets.** Copy `secrets.yaml.example` to `secrets.yaml` (next
   to your ESPHome configs) and fill in your Wi-Fi credentials.

4. **Edit `luke-roberts-lamp.yaml`** — replace at minimum:
   - `lr01_mac` with your lamp's MAC
   - `encryption_key` with a fresh 32-byte base64 API encryption key
   - `ota_encryption_key` with an OTA password
   - `ap_fallback_pw` with a fallback AP password

5. **Compile and flash** via the ESPHome dashboard or CLI:

   ```bash
   esphome run luke-roberts-lamp.yaml
   ```

6. **Adopt in Home Assistant.** The device should appear automatically via
   mDNS. If not, add it by hostname.

## Command reference

All payloads are written to characteristic
`44092842-0567-11E6-B862-0002A5D5C51B` on service
`44092840-0567-11E6-B862-0002A5D5C51B`. Every frame starts with `A0` followed
by API-version `01` or `02`.

| Button in HA                     | Opcode | Payload (hex)         | Notes                                          |
| -------------------------------- | :----: | --------------------- | ---------------------------------------------- |
| Ping                             | 00     | `A0 02 00`            | Returns `00 VV` – lamp's max API version       |
| Lamp Off                         | 05     | `A0 02 05 00`         | Scene id 0 = Off                               |
| Default Scene                    | 05     | `A0 02 05 FF`         | Power-up scene                                 |
| Scene 1 / 2                      | 05     | `A0 02 05 01` / `02`  | User-defined scenes from the Luke Roberts app  |
| Brightness 50 %                  | 03     | `A0 01 03 32`         | Absolute, percent 0..100                       |
| Color Temperature 3000 K         | 04     | `A0 01 04 0B B8`      | Kelvin big-endian, clamped to 2700..4000       |
| Next Brighter Scene              | 06     | `A0 02 06 01`         | Signed direction, +1                           |
| Next Dimmer Scene                | 06     | `A0 02 06 FF`         | Signed -1                                      |
| Brighter (+20 %) / Dimmer (-20 %)| 08     | `A0 02 08 78` / `50`  | Multiplicative relative brightness             |
| Immediate: Downlight 50 % 3000 K | 02     | `A0 01 02 02 0000 0BB8 80` | Flags=downlight, duration=infinite, BR=128 |

See the [API PDF](/docs/Lamp%20Control%20API%20v1.3.pdf) for the full frame layout,
response codes, and the Immediate Light uplight (HSB) sub-packet.

## Response decoding

Each write triggers a notification carrying the status byte:

| Byte | Meaning           |
| ---- | ----------------- |
| `00` | Success           |
| `81` | Invalid Params    |
| `84` | Invalid Id        |
| `87` | Invalid Version   |
| `BC` | Bad Command       |
| `FC` | Forbidden (lamp has a security code set via the app) |

The decoded response is published live to the HA text sensor
**`Luke Roberts API Response`** (diagnostic category).

## Connection behaviour

- Luke Roberts lamps with firmware ≥ 1.1.1 **drop the BLE link after ~8 s of
  inactivity**.
- `auto_connect: true` makes ESPHome reconnect automatically whenever the
  lamp is in range.
- A ping keepalive (`A0 02 00`, every 6 s) runs for 30 s after the last user
  command, so follow-up commands complete in < 100 ms without having to
  re-establish the link. Once the user goes idle the pings stop and the lamp
  is free to sleep.
- If the link happens to be down when you press a button, `lr_send` transparently
  reconnects before writing — just with a one-off latency of 1–2 s.

## File layout

```
.
├── README.md
├── LICENSE
├── luke-roberts-lamp.yaml      # the ESPHome config
├── secrets.yaml.example        # Wi-Fi credential template
└── docs/
    └── Lamp-Control-API-v1.3.pdf
```

## Extending

Ideas that fit cleanly on top of this base:

- **Scene list as a `select` entity** — use `A0 01 01 II` (Query Scene) to
  walk the scene list on connect and populate a dropdown with real names.
- **Unified `light.template` entity** — expose brightness + color-temperature
  through a single HA light instead of separate buttons (uses `03` + `04`
  under the hood).
- **Long-press / click-detection passthrough** if you mount the ESP near the
  lamp and want it to mirror the click events.

Pull requests welcome.

## Credits

- Luke Roberts GmbH — for publishing a clean, documented external control API.
- The ESPHome project — for making this kind of glue trivial to write.

## License

MIT – see [LICENSE](LICENSE).

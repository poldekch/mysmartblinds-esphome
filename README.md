# mysmartblinds-esphome

An ESPHome package that gives you local, cloud-free control of **first-gen
MySmartBlinds / Tilt motors** (the ones that advertise as `SmartBlind_DFU`).
You can add it to any existing ESP32 node, including one that already works as
a Bluetooth proxy, and it drives up to two blinds per device.

## What it does

For each blind you include, the package creates:

- a **cover** entity in Home Assistant (`device_class: blind`) with open,
  close and tilt
- a `ble_client` that connects **only on demand**. It stays disconnected while
  idle, so the radio stays free for scanning and proxying
- a send script that connects, authenticates with the blind's key, writes the
  target position, disconnects, and retries up to 3 times. If a new command
  arrives during a move, it replaces the old one
- **battery and sensor entities**, read from the motor's indications on every
  connection (see [Sensors](#sensors)): battery %, solar charging, the
  motor's reported position, a raw light level, and raw diagnostic values
- a **Refresh sensors** button and a **Sensor refresh interval** setting
  (default 10 min), so the sensors stay current between moves

Protocol (all characteristics share the base
`xxxxxxxx-1212-efde-1600-785feabcd123`):

| Characteristic | Direction | Meaning |
|---|---|---|
| `00001409` | write | pairing key; must be written first on every connection |
| `00001403` | write | one position byte: `100` flat/open … `200` closed; tilt maps as `200 - tilt*100` |
| `00001403` | indicate | a position byte (one motor reported `0x63` = 99); not yet known to follow wand or schedule moves |
| `00001402` | indicate | status; bytes 0–3 are a little-endian bitfield: `0x00040000` solar charging, `0x00080000` USB charging |
| `00001651` | indicate | 14 bytes: battery %, reserved, then six little-endian `uint16`: battery voltage, battery charge, solar-panel voltage, interior temperature, battery temperature, raw light |

Key and position writes are verified on hardware. The indications come from
a capture of the official Android app; as in the app, they are enabled only
after the key has been written. Reading `00001402`/`00001404` directly does
not return this data. Because it isn't yet known whether the `00001403`
readback tracks moves made with the wand or the motor's schedules, the cover
remains `assumed_state` / optimistic.

### Designed to drop into an existing device

- **Package-local variables.** Settings are passed as `vars:` on the package,
  not as global `substitutions:`. Each copy of the package has its own scope,
  so two blinds can't overwrite each other's MAC or key.
- **Prefixed IDs.** Every global, script, client and cover ID is prefixed with
  `blind_id`, so the blinds are fully independent and don't clash with your
  own IDs.
- **No BLE settings imposed.** The package declares `esp32_ble_tracker: {}`
  only because `ble_client` requires it. Your device config controls scan
  parameters, `bluetooth_proxy`, `esp32_ble`, and so on, and it always takes
  precedence over the package.
- **No `esphome:` hooks.** The motor needs a RANDOM address type. The package
  sets it right before each connect rather than in `esphome: on_boot`, so it
  doesn't touch your device's `esphome:` block.

## Getting MAC and key

Export every blind's MAC and key from the cloud with
[smartblinds-ble](https://github.com/benbalter/smartblinds-ble), while the
cloud is still online:

```sh
pip install "smartblinds-ble[cloud]"
smartblinds-import-cloud -o smartblinds-keys.json
```

Each entry in `smartblinds-keys.json` looks like
`{"name": ..., "mac": "AA:BB:CC:DD:EE:FF", "key": "0a1b2c3d", ...}`. Copy `mac`
and `key` **as they are** into the `vars:` block for that blind. The package
converts them itself:

- **MAC.** The dump has the cloud MAC, whose bytes are in the reverse order
  of the address the motor advertises. The package reverses it before
  connecting, so `AA:BB:CC:DD:EE:FF` is dialled as `FF:EE:DD:CC:BB:AA`.
  Don't reverse it yourself.
- **Key.** The hex string is split into bytes in the same order: `0a1b2c3d`
  is sent as `[0x0A, 0x1B, 0x2C, 0x3D]`, which is the byte list the forum
  example writes by hand. That list format (`"0x0A, 0x1B, 0x2C, 0x3D"`) is
  accepted as well. The key's length varies between blinds; send all of it.

| var          | meaning                                                  |
|--------------|----------------------------------------------------------|
| `blind_id`   | unique slug (`a-z0-9_`) used for all IDs; must differ per blind |
| `blind_name` | friendly name of the cover in Home Assistant             |
| `blind_mac`  | `mac` from the keys dump, unchanged, e.g. `"AA:BB:CC:DD:EE:FF"` |
| `blind_key`  | `key` from the keys dump, unchanged, e.g. `"0a1b2c3d"`   |

To check the conversion, open the device logs while moving a blind. You should
see `Connecting to FF:EE:DD:CC:BB:AA (dump MAC AA:BB:CC:DD:EE:FF reversed)`
followed by `Sending N-byte key`. The first address should match the
`SmartBlind_DFU` device in a BLE scanner app such as nRF Connect.

## Sensors

Every connection to the blind (each move, each scheduled refresh, each press
of **Refresh sensors**) also reads its sensors. The values stay in Home
Assistant between connections.

| Entity (prefixed with `blind_name`) | Source | Default |
|---|---|---|
| Battery | `00001651` byte 0, % | enabled |
| Solar charging | `00001402` bit `0x00040000` | enabled |
| Reported position (raw) | `00001403` indication, 0–200 scale; not used by the cover (see above) | enabled |
| Light level (raw) | `00001651` last `uint16`, not calibrated to lux | enabled |
| Battery voltage (raw), Battery charge (raw), Solar panel voltage (raw), Interior temperature (raw), Battery temperature (raw) | `00001651` `uint16` fields, units not verified | disabled, diagnostic |
| USB charging | `00001402` bit `0x00080000` | disabled, diagnostic |
| Status flags | `00001402` bytes 0–3 as hex | disabled, diagnostic |
| Refresh sensors | button: connect now and read the sensors | diagnostic |
| Sensor refresh interval | minutes between sensor-only connections, `1`–`1440` (24 h) | `10`, config |

The disabled entities can be enabled from the device page in Home Assistant.

**Battery cost.** A refresh is a short BLE connection: connect, discover
services, write the key, receive the indications, disconnect, which takes a
few seconds. At the default 10 minutes that is 144 connections a day, roughly
5–10 minutes of connected radio time in total, on top of the advertising the
motor does anyway. On a solar-charged motor this should be small, but it
hasn't been measured; watch the Battery entity for a few days, and raise the
interval if it trends down. A move resets the timer, since it reads the
sensors too. A refresh never interrupts a move. If the blind is unreachable,
the next try waits a full interval.

Each refresh also holds one of the ESP32's BLE connections for those few
seconds (see [BLE connection slots](#ble-connection-slots)).

**Calibrating raw values.** Only battery % has known units. Once you have
worked out a conversion, redeclare the sensor by its ID in your device config
and add filters. ESPHome merges list entries that share an ID:

```yaml
sensor:
  - id: office_light            # <blind_id>_light
    filters:
      - multiply: 0.5           # your own calibration
    unit_of_measurement: lx
```

The IDs are `<blind_id>_battery`, `_light`, `_battery_voltage`,
`_battery_charge`, `_solar_voltage`, `_interior_temp`, `_battery_temp` and
`_reported_position`.

**Logs.** Each indication is logged at `DEBUG` with its raw bytes (for
example `0x1651: 64.00.…`), which helps when working out the units. If a motor
lacks one of the characteristics, the log says
`Characteristic 0x1651 not found` and those entities stay unknown; moving the
blind is unaffected.

## Usage

Add a `packages:` entry for each blind. ESPHome fetches the package from this
repository, so there is nothing to copy.

### One blind

```yaml
# ... your existing esphome/esp32/wifi/api/ota/logger config ...

esp32_ble_tracker:
  scan_parameters:
    active: true

bluetooth_proxy:
  active: true
  connection_slots: 2      # 1 blind + 2 proxy slots = 3 (the default limit)

packages:
  blind_office:
    url: https://github.com/poldekch/mysmartblinds-esphome
    ref: main
    refresh: 1d
    files:
      - path: blind.package.yaml
        vars:
          blind_id: office
          blind_name: "Office Blind"
          blind_mac: "aa:bb:cc:dd:ee:ff"
          blind_key: "000000000000"
```

Full file: [example-blind.yaml](example-blind.yaml)

### Two blinds on one ESP32

Add one package entry per blind, each with its own `blind_id`:

```yaml
# ... your existing esphome/esp32/wifi/api/ota/logger config ...

bluetooth_proxy:
  active: true
  connection_slots: 1      # 2 blinds + 1 proxy slot = 3 (the default limit)

packages:
  blind_left:
    url: https://github.com/poldekch/mysmartblinds-esphome
    ref: main
    refresh: 1d
    files:
      - path: blind.package.yaml
        vars:
          blind_id: bedroom_left
          blind_name: "Bedroom Left Blind"
          blind_mac: "aa:bb:cc:dd:ee:01"
          blind_key: "000000000000"

  blind_right:
    url: https://github.com/poldekch/mysmartblinds-esphome
    ref: main
    refresh: 1d
    files:
      - path: blind.package.yaml
        vars:
          blind_id: bedroom_right
          blind_name: "Bedroom Right Blind"
          blind_mac: "aa:bb:cc:dd:ee:02"
          blind_key: "111111111111"
```

Full file: [example-two-blinds.yaml](example-two-blinds.yaml)

`ref` can be a branch, a tag, or a commit. Pin it to a tag or commit if you
don't want updates to this repository to reach your devices automatically.
`refresh` sets how often ESPHome checks for a newer version.

### Using a local copy instead

If you'd rather not fetch from GitHub, copy `blind.package.yaml` next to your
device YAML and include it with the same vars:

```yaml
packages:
  blind_office: !include
    file: blind.package.yaml
    vars:
      blind_id: office
      blind_name: "Office Blind"
      blind_mac: "aa:bb:cc:dd:ee:ff"
      blind_key: "000000000000"
```

## BLE connection slots

An ESP32 can only hold a limited number of BLE connections at once. This limit
is `esp32_ble: max_connections`, and it defaults to **3**. Two things use
those connections:

- **each blind**: 1 connection (`ble_client`), held only while the blind
  moves or refreshes its sensors (a few seconds)
- **an active Bluetooth proxy**: `bluetooth_proxy: connection_slots`
  connections, which also defaults to 3

These must fit within the limit:

```
number of blinds + bluetooth_proxy connection_slots <= esp32_ble max_connections
```

With the defaults, a proxy uses all 3 connections and leaves none for a blind,
so you have to change one of the two settings:

| Setup                   | Option A: shrink the proxy           | Option B: raise the limit                     |
|-------------------------|--------------------------------------|-----------------------------------------------|
| 1 blind + active proxy  | `bluetooth_proxy: connection_slots: 2` | `esp32_ble: max_connections: 4` (proxy keeps 3) |
| 2 blinds + active proxy | `bluetooth_proxy: connection_slots: 1` | `esp32_ble: max_connections: 5` (proxy keeps 3) |
| 1–2 blinds, no proxy, or a passive proxy (`active: false`) | nothing to change | nothing to change |

Option A is the safe default. Option B keeps full proxy capacity, but every
extra connection uses more RAM, which can be tight on boards that also run
Wi-Fi and many other components. If the numbers don't add up, ESPHome refuses
the config when it validates it.

Option B looks like this:

```yaml
esp32_ble:
  max_connections: 5

bluetooth_proxy:
  active: true            # connection_slots stays at the default 3
```

## Notes

- **Framework.** ESP-IDF is recommended, and an active Bluetooth proxy needs
  it.
- **Updating from an earlier version.** Nothing changes in your config; the
  sensor, button and number entities simply appear after the update.
- **Unique `blind_id`.** If two packages use the same `blind_id`, ESPHome
  merges them into one blind instead of reporting an error, so give each blind
  its own.

## Credits

All of the protocol work, key extraction and the original ESPHome
configuration come from the Home Assistant community guide
[Local control of first-gen MySmartBlinds / Tilt motors with ESPHome (no cloud)](https://community.home-assistant.io/t/local-control-of-first-gen-mysmartblinds-tilt-motors-with-esphome-no-cloud/1026182)
and its `smartblinds_ble` tooling. This repository only repackages that work
so it can be used as a multi-instance add-on to existing ESPHome devices.

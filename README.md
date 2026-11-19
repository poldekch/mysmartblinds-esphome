# mysmartblinds-esphome

An ESPHome package that gives you local, cloud-free control of **first-gen
MySmartBlinds / Tilt motors** (the ones that advertise as `SmartBlind_DFU`).
You can add it to any existing ESP32 node, including one that already works as
a Bluetooth proxy, and it drives up to two blinds per device.

## What it does

For each blind you include, the package creates:

- a **cover** entity in Home Assistant (`device_class: blind`) with open,
  close and tilt, whose state follows the position the motor reports,
  including moves made with the wand (see [Positions](#positions))
- configurable **open and close positions** ("smart" open/close, either
  closing direction) and buttons for the three fixed positions
- a `ble_client` that connects **only on demand**. It stays disconnected while
  idle, so the radio stays free for scanning and proxying
- a send script that connects, authenticates with the blind's key, writes the
  target position, disconnects, and retries up to 3 times. If a new command
  arrives during a move, it replaces the old one
- **battery and sensor entities**, read from the motor's indications on every
  connection (see [Sensors](#sensors)): battery %, solar charging, the
  motor's reported position, a raw light level, and diagnostic values
- a **Refresh sensors** button and a **Sensor refresh interval** setting
  (default 10 min), so the sensors stay current between moves

Protocol (all characteristics share the base
`xxxxxxxx-1212-efde-1600-785feabcd123`):

| Characteristic | Direction | Meaning |
|---|---|---|
| `00001409` | write | pairing key; must be written first on every connection |
| `00001403` | write | one position byte: `100` flat/open, `200` closed one way, `0` closed the other way (the `0` side is from older community notes, not yet verified) |
| `00001403` | read / indicate | the motor's position on the same scale; follows wand moves. Right after a move it returns the target until the motor finishes |
| `00001402` | indicate | status; bytes 0–3 are a little-endian bitfield: `0x00040000` solar charging, `0x00080000` USB charging |
| `00001651` | indicate | 14 bytes: battery %, reserved, then six little-endian 16-bit values: battery voltage, battery charge, solar-panel voltage, interior temperature, battery temperature, raw light |

Key and position writes are verified on hardware. The indications come from
a capture of the official Android app; as in the app, they are enabled only
after the key has been written. Reading `00001402`/`00001404` directly does
not return this data. The `00001403` readback was checked on hardware: it
reported 199 after closing to 200, and 131 after a move with the wand.

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

## Positions

The motor takes one position byte from `0` to `200`. `100` is flat (fully
open); `200` closes the slats one way and, according to older community
notes, `0` closes them the other way. Which of the two is "top of the slats
toward the room" and which is "toward the window" depends on how the blind is
mounted, so try both and rename the buttons in Home Assistant if you like.

| Entity (prefixed with `blind_name`) | What it does | Default |
|---|---|---|
| the cover's **open** | moves to **Open position** | |
| the cover's **close** | moves to **Close position** | |
| the cover's **tilt** slider | 100% = flat, 0% = closed toward the side of **Close position** | |
| Open position | "smart open": e.g. `80` or `120` to angle the slats for more light | `100`, config |
| Close position | "smart close": `200` for one direction, `0` (or `1`) for the other | `200`, config |
| Close (0 side), Open flat, Close (200 side) | buttons: move straight to `0`, `100`, `200`, ignoring the settings | |
| Reported position (raw) | the position the motor reports, `0`–`200` | enabled |

The cover's state comes from the motor's reported position, converted to
tilt as `1 - |position - 100| / 100`: 100% at flat, 0% at either closed end,
and "closed" within 5 steps of an end. It updates on every connection, so a
wand move shows up at the next refresh. A move from Home Assistant shows the
target immediately. Open and close stay available at all times, since the
blind can be moved behind Home Assistant's back.

## Sensors

Every connection to the blind (each move, each scheduled refresh, each press
of **Refresh sensors**) also reads its sensors. The values stay in Home
Assistant between connections.

| Entity (prefixed with `blind_name`) | Source | Default |
|---|---|---|
| Battery | `00001651` byte 0, % | enabled |
| Solar charging | `00001402` bit `0x00040000` | enabled |
| Reported position (raw) | `00001403`, 0–200 (see [Positions](#positions)) | enabled |
| Light level (raw) | `00001651` last value, not calibrated to lux | enabled |
| Battery voltage, Solar panel voltage | `00001651`, assumed mV, shown in V | disabled, diagnostic |
| Interior temperature, Battery temperature | `00001651`, assumed 0.1 °C, shown in °C | disabled, diagnostic |
| Battery charge (raw) | `00001651`, unknown units (so far always equal to the solar panel value) | disabled, diagnostic |
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

**Units.** Only battery % is known for certain. Volts and °C are inferred
from readings that look plausible (3.39 V battery, 19.6 °C inside, 24.7 °C
battery); treat them as unverified. To correct or calibrate a value,
redeclare the sensor by its ID in your device config and add filters.
ESPHome merges list entries that share an ID:

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
blind is unaffected. `0x1403` is also read directly on every connection, and
its log line says whether a value came from that `read` or from an
`indication`. A connection that ends without hearing from a characteristic
logs `No data this connection from: …`. Once per boot the log also names
the service each characteristic belongs to (`0x1651 is in service …`).

ESPHome itself logs a few warnings on every connection that are harmless
here: `ble_sensor` reports `Disconnected!` because the blind disconnects on
purpose, and `No sensor characteristic found at service 00001650…` because
`0x1651`'s service isn't known yet (the package finds it by UUID anyway).
The related `ble_client.sensor set Warning flag` line comes from the same
cause. To hide the `ble_sensor` ones, add this to your device config:

```yaml
logger:
  logs:
    ble_sensor: ERROR
```

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

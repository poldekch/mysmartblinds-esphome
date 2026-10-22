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

Protocol (verified on hardware): the key goes to characteristic `00001409`,
then a single position byte goes to `00001403` (service `00001400`). The
position ranges from `100` (flat/open) to `200` (closed), and tilt maps as
`200 - tilt*100`. The motor has no reliable position feedback, so the cover is
`assumed_state` / optimistic.

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

You need each blind's **advertised** MAC (the cloud MAC with its bytes
reversed) and its BLE key as hex. The generator from the original project
prints both:

```sh
python -m smartblinds_ble.tools.control --esphome smartblinds-keys.json
```

Put its output into the `vars:` block for each blind (`blind_mac`,
`blind_key`).

| var          | meaning                                                  |
|--------------|----------------------------------------------------------|
| `blind_id`   | unique slug (`a-z0-9_`) used for all IDs; must differ per blind |
| `blind_name` | friendly name of the cover in Home Assistant             |
| `blind_mac`  | advertised (reversed) MAC, e.g. `"aa:bb:cc:dd:ee:ff"`    |
| `blind_key`  | BLE key as hex, exactly as in the keys JSON              |

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
  moves
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
- **Unique `blind_id`.** If two packages use the same `blind_id`, ESPHome
  merges them into one blind instead of reporting an error, so give each blind
  its own.

## Credits

All of the protocol work, key extraction and the original ESPHome
configuration come from the Home Assistant community guide
[Local control of first-gen MySmartBlinds / Tilt motors with ESPHome (no cloud)](https://community.home-assistant.io/t/local-control-of-first-gen-mysmartblinds-tilt-motors-with-esphome-no-cloud/1026182)
and its `smartblinds_ble` tooling. This repository only repackages that work
so it can be used as a multi-instance add-on to existing ESPHome devices.

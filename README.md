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
- **silent moves** that step the slats slowly over a configurable time, so the
  motor doesn't wake anyone (see [Silent moves](#silent-moves)), plus Home
  Assistant actions for fast and silent moves to any position
- a `ble_client` that connects **only on demand**. It stays disconnected while
  idle, so the radio stays free for scanning and proxying
- a send script that connects, authenticates with the blind's key, writes the
  target position, disconnects, and retries up to 3 times. A new command that
  arrives during a move replaces it, on the same connection if it is still open
- **battery and sensor entities**, read from the motor's indications on every
  connection (see [Sensors](#sensors)): battery %, solar charging, the
  motor's reported position, a raw light level, and diagnostic values
- a **Refresh sensors** button and a **Sensor refresh interval** setting
  (default 10 min), so the sensors stay current between moves
- the motor's own **too-hot mode**, switchable from Home Assistant: above a
  set temperature the motor closes by itself, even with the ESP offline (see
  [Too-hot mode](#too-hot-mode))
- **clock sync**: the motor's clock is set from Home Assistant's time on the
  first connection after boot and every 12 hours

Protocol (all characteristics share the base
`xxxxxxxx-1212-efde-1600-785feabcd123`):

| Characteristic | Direction | Meaning |
|---|---|---|
| `00001409` | write | pairing key; must be written first on every connection. A 6-byte key is sent followed by `0x01`, as the official app does |
| `00001409` | read | on an unpaired motor, the official app reads its new key here (see the **Read pairing key** button) |
| `00001403` | write | one position byte: `100` flat/open, `0` and `200` closed in opposite directions |
| `00001403` | read / indicate | the motor's position on the same scale; follows wand moves. Right after a move it returns the target until the motor finishes |
| `00001402` | read / indicate | status; bytes 0–3 are a little-endian bitfield, e.g. `0x80000000` key accepted, `0x00010000` calibrated, `0x00040000` solar charging, `0x00200000` battery lockout. Read directly after the key on every connection |
| `00001651` | indicate | 14 bytes: battery %, reserved, then six little-endian 16-bit values: battery voltage, battery charge, solar-panel voltage, interior temperature, battery temperature, raw light |

Key and position writes are verified on hardware. The indications come from
a capture of the official Android app; as in the app, they are enabled only
after the key has been written. The `00001403` readback was checked on
hardware: it reported 199 after closing to 200, and 131 after a move with the
wand.

The full protocol as the official app uses it (status bits, pairing,
calibration, the motor's own schedule and sun tracking, energy savings) is
written up in [docs/PROTOCOL.md](docs/PROTOCOL.md), with each part marked as
verified on hardware or taken from the app only.

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
  A 6-byte key goes out with `0x01` appended, as the official app sends it
  (the log then says `Sending 7-byte key`); keys of other lengths are sent
  unchanged.

| var          | meaning                                                  |
|--------------|----------------------------------------------------------|
| `blind_id`   | unique slug (`a-z0-9_`) used for all IDs; must differ per blind |
| `blind_name` | friendly name of the cover in Home Assistant             |
| `blind_mac`  | `mac` from the keys dump, unchanged, e.g. `"AA:BB:CC:DD:EE:FF"` |
| `blind_key`  | `key` from the keys dump, unchanged, e.g. `"0a1b2c3d"`   |
| `sticky_blind` | optional, default `false`; `true` for a blind that jams when closed (see [Sticky blinds](#sticky-blinds)) |

To check the conversion, open the device logs while moving a blind. You should
see `Connecting to FF:EE:DD:CC:BB:AA (dump MAC AA:BB:CC:DD:EE:FF reversed)`
followed by `Sending N-byte key`. The first address should match the
`SmartBlind_DFU` device in a BLE scanner app such as nRF Connect.

## Positions

The motor takes one position byte from `0` to `200`. `100` is flat (fully
open), and the two ends close the slats in opposite directions. Verified on
hardware:

| Position | Slats |
|---|---|
| `0` | closed, room edge of the slats **down** |
| `100` | flat, fully open |
| `200` | closed, room edge of the slats **up** |

If your blind is mounted differently, check which way each end goes and
rename the buttons in Home Assistant if you like.

| Entity (prefixed with `blind_name`) | What it does | Default |
|---|---|---|
| the cover's **open** | moves to **Open position** | |
| the cover's **close** | moves to **Close position** | |
| the cover's **tilt** slider | 100% = flat, 0% = closed toward the side of **Close position** | |
| Open position | "smart open": e.g. `80` or `120` to angle the slats for more light | `100`, config |
| Close position | "smart close": `200` closes room edge up, `0` room edge down | `200`, config |
| Close (0 side), Open flat, Close (200 side) | buttons: move straight to `0`, `100`, `200`, ignoring the settings | |
| Reported position (raw) | the position the motor reports, `0`–`200` | enabled |

The cover's state comes from the motor's reported position, converted to
tilt as `1 - |position - 100| / 100`: 100% at flat, 0% at either closed end,
and "closed" within 5 steps of an end. It updates on every connection, so a
wand move shows up at the next refresh. A move from Home Assistant shows the
target immediately. Open and close stay available at all times, since the
blind can be moved behind Home Assistant's back.

### Sticky blinds

Some motors occasionally jam in the closed position: Open does nothing, but a
small further move toward closed frees them. Set `sticky_blind: true` in that
blind's `vars:` (other blinds are unaffected):

```yaml
        vars:
          blind_id: office
          blind_name: "Office Blind"
          blind_mac: "aa:bb:cc:dd:ee:ff"
          blind_key: "000000000000"
          sticky_blind: true
```

With it enabled:

- **Closing stops 3 steps short of each end**: `200` becomes `197` and `0`
  becomes `3`. This applies to the cover's close, the tilt slider and the
  Close buttons. **Open position** and **Close position** only accept
  `3`–`197`.
- **Opening from closed nudges first.** If the blind is closed (within 5
  steps of an end) and a move opens it (Open, the tilt slider or **Open
  flat**), the package first moves it to full close on the side of **Close
  position** (`200` if it is 100 or more, else `0`), waits 1 second, then
  moves to the target. It all happens on one connection. The log shows
  `Sticky blind: nudging to 200 before moving to 100`.
- **No nudge when it's already open.** If the blind isn't closed, moves go
  straight to their target.
- **Visible in Home Assistant.** Every blind has a read-only **Sticky mode**
  entity in its Diagnostic section (disabled by default; enable it from the
  device page), on when `sticky_blind: true`. Change the setting in the YAML;
  it can't be toggled from Home Assistant.

The package decides whether the blind is closed from the last position it
knows: the motor's report from the most recent connection, or the target of
the latest move. Right after a reboot, before the first refresh, that
position is unknown, so the first Open doesn't nudge.

## Silent moves

The motor has no slow mode of its own (none is known, at least), so a silent
move is done by the ESP: it holds one connection and steps the position from
where the motor is now to the target, a little at a time.

| Entity / action (prefixed with `blind_name` / `blind_id`) | What it does | Default |
|---|---|---|
| Silent open | silent move to **Open position** | |
| Silent close | silent move to **Close position** | |
| Silent move duration | how long a silent move takes, whatever its distance: `30`–`1800` s | `100`, config |
| action `<blind_id>_move` | fast move to any `position` (`0`–`200`) | |
| action `<blind_id>_silent_move` | silent move to any `position` (`0`–`200`) | |

**How the steps are worked out.** Each silent move starts by reading the
motor's current position, then splits the distance into steps of at least 1
unit, at most one every 0.5 s, spread evenly over the duration. For example:

| Move | Duration | Steps |
|---|---|---|
| 80 → 197 | 100 s | 117 steps of 1, every 0.85 s |
| 80 → 197 | 30 s | 60 steps of about 2, every 0.5 s |
| 120 → 130 | 100 s | 10 steps of 1, every 10 s |

While it runs, the cover shows *opening* or *closing* and its tilt follows
each step. The log shows `Silent move 80 -> 197: 117 steps, 854 ms apart` at
the start and `Silent move finished at 197` at the end.

- **Interrupting.** Any other move (the cover, a button, an action) stops a
  silent move at the step it reached and takes over on the same connection.
- **Sticky blinds.** A silent open from closed does the quick 1-second nudge
  first, then the slow steps.
- **Dropped connection.** If the link drops, the package reconnects (up to 3
  tries) and continues from the motor's reported position.
- **Sensors.** Scheduled refreshes are skipped while a silent move runs; the
  sensors are read when it finishes.
- **Connection slot.** The blind's BLE connection stays open for the whole
  move (see [BLE connection slots](#ble-connection-slots)).

If the steps turn out noisier than one quick move, change `MIN_STEP_S` (the
0.5 s minimum between steps) in the package to try fewer, larger steps.

### In automations

The cover's own services (`cover.open_cover`, `cover.close_cover`,
`cover.set_cover_tilt_position`) are always fast. For silent moves, press the
buttons or call the actions. ESPHome names the actions
`esphome.<device name>_<blind_id>_move` and `..._silent_move`, with dashes in
the device name turned into underscores:

```yaml
# Silent open to the Open position setting
- action: button.press
  target:
    entity_id: button.master_bedroom_blind_silent_open

# Silent move to a specific angle
- action: esphome.bedroom_proxy_master_bedroom_silent_move
  data:
    position: 115

# Fast move to a specific angle
- action: esphome.bedroom_proxy_master_bedroom_move
  data:
    position: 80
```

Positions are raw (`0`–`200`, `100` = flat; `3`–`197` on a sticky blind) and
follow the same rules as the buttons. The package adds these actions to your
device's `api:` block; they merge with any actions you already have.

## Too-hot mode

The motor has its own "too hot" rule (called energy savings in the official
app). When the temperature measured by the motor itself goes above a set
point, it moves to a set position; once it is 3 °C cooler it can go back to
where it was. Because the motor runs the rule, it keeps working when the ESP
or Home Assistant is down.

| Entity (prefixed with `blind_name`) | What it does | Default |
|---|---|---|
| Too-hot mode | switch: turn the motor's rule on or off | off, config |
| Too-hot temperature | the set point, `16`–`40` °C (the app's range) | `28`, config |
| Too-hot position | where the blind goes when it's too hot (raw `0`–`200`; `3`–`197` on a sticky blind) | `200`, config |
| Too-hot resume | switch: go back to the previous position once 3 °C cooler | on, config |
| Too-hot active | the motor reports that the rule has taken over | |

- **How the settings reach the motor.** They are written about 3 seconds
  after you change one, and once after every boot of the ESP, since the
  motor's copy can't be read back. The same write carries the **Open
  position** and **Close position** as the motor's own "smart" open and
  close positions, which the wand may use.
- **Time needed.** The write includes a timestamp from Home Assistant's
  time, so it waits until the ESP has the time. The log line
  `Room settings: open …, close …, too-hot on at 28.0 C -> 200, resume`
  shows what was written.
- **The motor's own sensor.** The rule uses the motor's interior temperature
  (the **Interior temperature** entity). If that reads 0, as on a motor with
  a sensor fault, the rule never triggers.
- **Not yet verified on hardware.** This comes from the official app's code.
  To test it, set the temperature just below the room's and watch whether
  the blind moves by itself and **Too-hot active** turns on.

**Clock sync.** On the first connection after boot, then every 12 hours, the
package sets the motor's clock from Home Assistant's time (log:
`Setting the motor's clock: …`). If the motor reports its clock isn't set
(**Clock set** off, for example after a factory reset), it is set on the next
connection. The package adds a Home Assistant `time:` source per blind for
this; it doesn't change any time source you already have.

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
| Key accepted | `00001402` bit `0x80000000`: the motor accepted `blind_key` | diagnostic |
| Calibrated | `00001402` bit `0x00010000` | diagnostic |
| Solar panel | `00001402` bit `0x00020000`: a solar panel is connected | diagnostic |
| Battery lockout | `00001402` bit `0x00200000`: battery too low, the motor won't move (not even from the wand) until it charges | diagnostic |
| Over temperature | `00001402` bit `0x00400000` | diagnostic |
| Clock set | `00001402` bit `0x00100000`: the motor's clock is valid | diagnostic |
| Paired | `00001402` bit `0x20000000` | disabled, diagnostic |
| Status flags | `00001402` bytes 0–3 as hex | disabled, diagnostic |
| Refresh sensors | button: connect now and read the sensors | diagnostic |
| Read pairing key | button: on the next connection, read `00001409` before unlocking and show the result in **Pairing key read** | disabled, diagnostic |
| Pairing key read | first six bytes the button got, formatted like `blind_key` | disabled, diagnostic |
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
logs `No data this connection from: …`. If `0x1651` hasn't arrived shortly
after the key, it is read directly (`0x1651 not received yet; reading it`).
The status (`0x1402`) is read directly after the key too, and a warning is logged when it shows a refused
key, a battery lockout, over temperature or a motor that isn't calibrated.

**Read pairing key.** This button is for a motor that was factory reset or
whose key no longer works. It reads the motor's passkey characteristic
before unlocking; the log shows every byte (`0x1409 (read): …`) and
**Pairing key read** shows the first six, ready to paste into `blind_key`.
It is read only. On a paired motor the result may not be a usable key; the
official app reads it only while adding a new or reset blind.

ESPHome itself logs a couple of warnings on every connection that are
harmless here: `ble_sensor` reports `Disconnected!`, and `component` reports
`ble_client.sensor set Warning flag`, both because the package disconnects
on purpose after each move or refresh. To hide the `ble_sensor` ones, add
this to your device config:

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
  moves or refreshes its sensors (a few seconds; a silent move holds it for
  its whole duration)
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
- **Updating from an earlier version.** Nothing changes in your config; new
  entities simply appear after the update, and the actions appear under your
  device in Home Assistant.
- **Unique `blind_id`.** If two packages use the same `blind_id`, ESPHome
  merges them into one blind instead of reporting an error, so give each blind
  its own.

## Credits

All of the protocol work, key extraction and the original ESPHome
configuration come from the Home Assistant community guide
[Local control of first-gen MySmartBlinds / Tilt motors with ESPHome (no cloud)](https://community.home-assistant.io/t/local-control-of-first-gen-mysmartblinds-tilt-motors-with-esphome-no-cloud/1026182)
and its `smartblinds_ble` tooling. This repository only repackages that work
so it can be used as a multi-instance add-on to existing ESPHome devices.

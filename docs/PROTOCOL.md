# MySmartBlinds BLE protocol (first-generation `SmartBlind_DFU` motors)

This describes everything the official Android app (versions 2.4.5, 2.6.1 and
3.0.2, all identical at the protocol level) can do with a motor over
Bluetooth. It was worked out from the app's code, for interoperability with
motors the owner bought, and checked against real motors where marked.

**How sure each part is:**

- **verified**: seen working on a motor with this package.
- **app**: taken from the app's code but not yet tried on a motor.
- **guess**: inferred, with no direct evidence either way.

## Overview

The motor has one service, `00001400-1212-efde-1600-785feabcd123` (verified).
Every characteristic in the table below shares that base UUID. Only the short
ID changes: `0000xxxx-1212-efde-1600-785feabcd123`. Multi-byte numbers are
little-endian.

| Short ID | App name | Access | Purpose |
|---|---|---|---|
| `1401` | Name | write | blind name |
| `1402` | Status | read, write, indicate | 32 status/command bits ([Status](#1402-status)) |
| `1403` | Angle | read, write, indicate | position byte `0`–`200`, plus special bytes ([Position](#1403-position)) |
| `1404` | Version info | read, indicate | firmware and hardware versions ([Version](#1404-version)) |
| `1405` | Time | write | clock sync ([Time](#1405-time)) |
| `1407` | RxTx | write, indicate | command channel: calibration, keep-alive, firmware update ([RxTx](#1407-rxtx-command-channel)) |
| `1409` | Passkey | read, write, indicate | unlock key ([Pairing and unlocking](#pairing-and-unlocking)) |
| `140A` | Calibration | read, write | not used by the app beyond logging |
| `1501` | Schedule | read, write | the motor's own schedule ([Schedule](#1501-schedule)) |
| `1502` | DST / location | write | daylight saving rules, latitude, longitude, time zone ([Location](#1502-dst-and-location)) |
| `1503` | ACK | indicate | write acknowledgements (content not decoded by the app) |
| `1603` | Room settings | write | smart open/close positions and energy savings ([Room settings](#1603-room-settings)) |
| `1651` | Sensors | read, indicate | battery, voltages, temperatures, light ([Sensors](#1651-sensors)) |

**Timestamp prefix.** Several writes start with a 4-byte "reference time":
seconds since 2001-01-01 00:00 UTC, as an unsigned 32-bit little-endian
number. That is Unix time minus `978307200`. It is used by Name, Room settings
and Schedule, and appended to the rotation-direction status writes.

**Which indications the app turns on.** Right after connecting it enables
`1409`, `1404`, `1402` and `1407`. After a successful unlock it adds `1403`,
`1503` and `1651`.

## Pairing and unlocking

### Advertisement (app)

Motors advertise the service UUID `1400` together with service data under the
16-bit UUID `0x0ADD`. Byte 6 of that service data gives the pairing state:

| Byte 6 | Meaning |
|---|---|
| `0x80` | paired |
| `0x01` | "pulled": ready to pair. The app's add-blind screen looks for this. |
| `0x00` | not paired |

Two things are still unknown (guess):
- **What sets the "pulled" state.** The name suggests pulling the wand, maybe
  with a long pull or hold.
- **How the passkey is generated.** It is probably random, created by the
  motor at a factory reset.

### Pairing a new or reset motor (app)

1. Connect and enable indications on `1409`, `1404`, `1402` and `1407`.
2. **Read `1409`.** It returns 7 bytes. The first 6 are the passkey, so drop
   the last byte. The motor produces this key itself; neither the app nor the
   cloud creates it. The app saves it locally, and the cloud copy was only a
   backup.
3. **Clear calibration:** send `64 00` on `1407` (see [Calibration](#calibration)).
4. **Unlock:** write the passkey followed by `01` (7 bytes) to `1409`.
5. **Wait for the unlock result.** The motor sends a `1402` status indication
   with `IsPasskeyValid` (bit 31) set. `IsPasskeyInvalid` (bit 30) means the
   key was refused, and the app then disconnects.
6. **After the unlock** the app:
   - reads `1404`
   - writes the clock to `1405`
   - reads the schedule from `1501`
   - enables indications on `1403`, `1503` and `1651`
   - writes location and time zone to `1502`

### Unlocking on every connection (app, partly verified)

- **The app's format.** It writes the 6-byte passkey followed by `01`. If the
  stored key is longer than 6 bytes, it first drops the key's first byte.
- **What this package does.** It writes the key the app's way (6 bytes +
  `01`), which the motor accepts (verified). Earlier versions wrote only the
  6 key bytes, which also worked (verified).
- **Reading `1409` on a paired motor** returns `FF` × 7 (verified), so a
  paired motor doesn't give its key away. Whether an unpaired one does is
  still to be tested.
- **The unlock result.** The motor reports the outcome in `1402` (bit 31
  valid, bit 30 invalid) and, if refused, as a NACK on `1407` (see
  [NACK reasons](#nack-reasons)).

## 1402 Status

Bytes 0–3 form a 32-bit little-endian value. The low bits are commands (you set
them by writing). The upper bits report state (the motor sets them). A direct
read after the key returns 8 bytes, the flags followed by 4 zero bytes
(verified). A working, calibrated blind read `0xA0110000`: key valid, paired,
clock set, calibrated; `0xA0150000` while charging from solar.

After a factory reset the motor's clock is not set, and a blind in that state
reported only zeros (apart from battery %) on `1651`. Its readings came back
within a day of this package setting the clock (verified on one blind), so the
motor may only measure while its clock is valid (guess).

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 0 | `0x00000001` | IsReversed | rotation reversed. Write it set or cleared, followed by a timestamp (see below). |
| 1 | `0x00000002` | IsBonding | bonding in progress |
| 2 | `0x00000004` | Calibrate | command: legacy calibration (protocol < 3) |
| 3 | `0x00000008` | Identify | command: identify the blind (guess: blink or jog) |
| 13 | `0x00002000` | ResetPasskey | command: reset the passkey |
| 14 | `0x00004000` | ResetFlash | command: **factory reset** |
| 15 | `0x00008000` | Disconnect | command: disconnect |
| 16 | `0x00010000` | IsCalibrated | calibrated. When clear, the app shows "needs calibration". |
| 17 | `0x00020000` | HasSolar | solar panel present according to the app's name, but it stayed `0` on two blinds with fitted panels, at night and while charging in daylight (verified). Meaning unclear. |
| 18 | `0x00040000` | IsChargingSolar | charging from solar (verified) |
| 19 | `0x00080000` | IsChargingUSB | charging from USB (verified) |
| 20 | `0x00100000` | IsTimeValid | the clock has been set |
| 21 | `0x00200000` | UnderVoltageLockout | **battery too low: the motor locks itself** |
| 22 | `0x00400000` | IsOverTemp | over temperature |
| 23 | `0x00800000` | TempOverride | energy savings has taken over ([Room settings](#1603-room-settings)) |
| 29 | `0x20000000` | IsPaired | paired |
| 30 | `0x40000000` | IsPasskeyInvalid | the last key write was refused |
| 31 | `0x80000000` | IsPasskeyValid | unlocked |

**Writing a command (app):**
1. Read the current 4-byte status value.
2. OR in the command bit. To return to normal rotation, clear bit 0 instead.
3. Write the result back as 4 bytes.
4. For the two rotation commands only, add the 4-byte timestamp after it.

The app sends factory reset and passkey reset only while unlocked, then
forgets its stored key.

## 1403 Position

- **Position byte (verified):**
  - `0` = closed with the room edge of the slats down
  - `100` = flat
  - `200` = closed with the room edge up
- **Reads and indications (verified):** they report the motor's current
  position. After a factory reset, or before the key is accepted, they report
  `255`.
- **Bytes above 200 are commands (app).** The package must never write them by
  accident:

| Byte | App name | Meaning |
|---|---|---|
| `0xF0` | Step DOWN | jog one step down (legacy calibration) |
| `0xF1` | Step UP | jog one step up (legacy calibration) |
| `0xF2` | Adjust DOWN | save the current position as the "down" end (legacy) |
| `0xF3` | Adjust UP | save the current position as the "up" end (legacy) |
| `0xF4` | Adjust CENTER | save the current position as flat (legacy) |

The app's fixed positions are `0`, `50`, `100`, `150` and `200`.

## 1404 Version

20 bytes:

| Offset | Field |
|---|---|
| 0–3 | hash (u32) |
| 4 / 5 | hardware type major / minor |
| 6 | hardware version |
| 7 | **protocol version.** Calibration over `1407` needs **3 or higher**; older motors use the legacy calibration in `1402`/`1403`. |
| 8 / 9 | package version major / minor |
| 10 / 11 | MCU application major / minor |
| 12 / 13 | MCU bootloader major / minor |
| 14 / 15 | BLE application major / minor |
| 16–17 | BLE stack ID (u16) |
| 18 / 19 | BLE DFU major / minor |

The app retries the read up to 6 times, 200 ms apart, if any field reads 0.

Seen on two first-generation motors (verified): hardware 13 (type 2.1),
protocol 3, MCU 2.3 (bootloader 1.2), BLE DFU 3.0, BLE stack `0x0065`, and a
package version of 0.0. The BLE application was 2.0 on an updated motor and
1.19 on one that had missed an update. The app's public "v2" firmware list
applies only to hardware 21.

## 1405 Time

6 bytes:
- bytes 0–1: minute of the week (u16), counting from Sunday 00:00, in local
  **standard** time (the daylight saving hour is removed first)
- byte 2: year − 2000
- byte 3: month (1–12)
- byte 4: day of the month
- byte 5: seconds, with bit 7 set while daylight saving time is in effect

The app writes this after every unlock. This package writes it on the first
connection after boot, then every 12 hours, and right away when `IsTimeValid`
is clear.

## 1407 RxTx (command channel)

You write commands to `1407`, and the motor answers with indications on
`1407`. The first byte is the command. The app resends a calibration command
every 250 ms, up to 4 times, until an answer arrives.

| First byte | Use |
|---|---|
| `0x00` / `0x01` | answer: NACK / ACK. Byte 2 is the command being answered. |
| `0x31` | debug output |
| `0x42` | keep-alive ping: the app writes `42 00` while it holds a connection |
| `0x50`–`0x55` | MCU firmware update (file start, block start, block data, validate, finalize, abort): **do not use** |
| `0x56`–`0x58` | logging (init, get info, get data) |
| `0x60`–`0x64` | calibration ([below](#calibration)) |
| `0xDF` | switch to BLE firmware update (DFU) mode: **do not use** |

### NACK reasons

| Code | Meaning |
|---|---|
| `0x00` | invalid command |
| `0x10` | passkey invalid |
| `0x11` | passkey not ready |
| `0x20` | file error |
| `0x21` | file busy |

## Calibration

These commands apply to protocol version 3 and above (app). Calibration points
are `0` direction, `1` TOP, `2` MIDDLE and `3` BOTTOM. TOP is closed with the
room edge up (position `200`), MIDDLE is open/flat (`100`) and BOTTOM is closed
the other way (`0`).

| Command | Bytes | Answer |
|---|---|---|
| CLEAR | `64 00` | ACK naming `0x64` |
| GET point | `60 01 <point>` | `60 .. <point> <state>`, where state `0` = uncalibrated and `1` = calibrated |
| GET direction | `60 01 00` | `60 .. 00 <dir>`, where dir `0` = normal and `1` = reversed |
| SET point calibrated | `61 02 <point> 01` | `61 .. <point>` |
| SET direction | `61 02 00 <dir>` | `61 .. 00` |
| MOVE (jog) | `62 01 <speed>` | none. Speed is a signed byte, about −95..+95; the sign gives the direction. |
| MOVE TO point | `63 01 <point>` | ACK naming `0x63` |

This package's **Check calibration** button sends the four GET commands
(direction, TOP, MIDDLE, BOTTOM) and logs every answer on `1407`. Verified on
two motors: each GET is answered within about 0.3 s by a 20-byte indication,
`60 02 <point> <value>` padded with `FF`, e.g. `60 02 00 00` (direction
normal) and `60 02 01 01` (TOP calibrated). A calibrated motor answered
`0 / 1 / 1 / 1`.

### The app's calibration screens, in order

Each point is calibrated the same way:
1. Move to the point.
2. Ask whether it is calibrated.
3. Let the user jog it into place.
4. Save it as calibrated.

| Step | Commands |
|---|---|
| 1. Start | CLEAR |
| 2. TOP (closed, room edge up) | GET direction · MOVE TO TOP · GET TOP · jog · SET TOP |
| 3. BOTTOM (closed, room edge down) | MOVE TO BOTTOM · GET BOTTOM · jog · SET BOTTOM |
| 4. MIDDLE (open) | MOVE TO MIDDLE · GET MIDDLE · jog · SET MIDDLE |

**Jogging.** While the user holds the slider, the app sends MOVE `62 01
<speed>` every 200 ms. It skips speeds within ±5 of zero, and stops sending
when the slider is released.

**Reversing.** A "reversed" switch sends SET direction with `0` (normal) or
`1` (reversed).

**Refusals.** The app won't start calibrating if the battery is low, or if the
protocol version is below 3; in that case it asks for a firmware update first.

### Legacy calibration (protocol below 3, app)

1. Set Calibrate in `1402`.
2. Jog with `0xF0`/`0xF1` on `1403`.
3. Save each end with `0xF2` (down), `0xF3` (up) and `0xF4` (flat) on `1403`.

## 1501 Schedule

The motor runs its own schedule, with no app or ESP connected (app). The
schedule is at most 136 bytes:
- bytes 0–3: the time it was last updated (a timestamp)
- then up to 33 events of 4 bytes each, little-endian u32
- the list ends at the first `0xFFFFFFFF`

The motor clears or rewrites the remaining bytes. Each event:

| Bits | Field |
|---|---|
| 31–24 | position `0`–`200` |
| 23 | enabled |
| 22–16 | days: bit 16 = Sunday … bit 22 = Saturday |
| 15 | sun tracking |
| 14–11 | gradual tilt, in minutes (the app only ever sets `1` = on) |
| 10–0 | minute of the day, `0`–`1439` |

**Sun tracking (app).** With bit 15 set, the time is stored as a "reference"
time: the equivalent time on day 80 of the year (about 21 March), in local
standard time. Each day the motor maps it onto the real sunrise and sunset
for the latitude and longitude in `1502`:
- **Near sunrise or sunset** (within ±60 min): the event keeps its fixed
  offset from sunrise or sunset.
- **Anywhere else:** the time scales in proportion between solar noon and
  sunset (or sunrise).
- **Wrapping past midnight:** if the converted time crosses midnight, the
  event's day mask shifts by one day.

**Gradual tilt (app).** This field suggests the motor can make a slow move by
itself. Whether values other than `1` work is a guess.

After a schedule write, the app reads `1501` back and compares the first 4
bytes to check that it was stored.

## 1502 DST and location

12 bytes (app):

| Offset | Field |
|---|---|
| 0–3 | DST start: month, week, day, hour |
| 4–7 | DST end: month, week, day, hour |
| 8 | latitude, whole degrees (signed byte) |
| 9–10 | longitude, whole degrees (s16) |
| 11 | base time zone offset in **quarter hours** (UTC+1 → `4`) |

The app writes this after every unlock if it knows the phone's location. The
exact encoding of week and day is not decoded yet.

## 1603 Room settings

The 4-byte timestamp followed by (app):

| Offset | Field |
|---|---|
| 0 | smart open position (`0`–`200`) |
| 1 | smart close position (`0`–`200`) |
| 2–3 | energy savings temperature set point, s16 in 0.1 °C: `160`–`400` (16.0–40.0 °C), default `280` |
| 4 | energy savings position (default `200`) |
| 5 | flags: bit 0 enabled, bit 1 resume afterwards (default resume on, savings off) |
| 6 | hysteresis in 0.1 °C (`30` = 3.0 °C) |
| 7 | `0` |

Bytes 2–7 are optional; the app sends only bytes 0–1 when energy savings isn't
set. Energy savings is the motor's own "too hot" rule (app): above the set
point it moves to the energy savings position, and below the set point minus
the hysteresis it goes back if "resume" is on. While it is in control, status
bit 23 (TempOverride) is set. That the wand toggles between the smart open and
smart close positions is a guess.

This package always writes all 8 bytes, with the enabled flag cleared when
too-hot mode is off: once after each ESP boot, and whenever a setting changes.

## 1651 Sensors

14 bytes; the app reads the 16-bit fields as **signed**:

| Offset | Field | Unit (this package) |
|---|---|---|
| 0 | battery % | % (verified) |
| 1 | reserved | |
| 2–3 | battery voltage | mV (guess) |
| 4–5 | battery charge | raw. Probably the charger's output in mV: about 3400–3550 while charging, equal to the solar voltage at night (verified readings) |
| 6–7 | solar panel voltage | mV (guess) |
| 8–9 | interior temperature | 0.1 °C (guess). Energy savings compares against this, and its set point is in 0.1 °C. |
| 10–11 | battery temperature | 0.1 °C (guess) |
| 12–13 | light | raw |

## Safety notes for implementers

- **Firmware update commands.** Never send `0xDF` or `0x50`–`0x55` on `1407`.
- **Position bytes.** Never write a position above `200` to `1403`; those
  bytes are commands.
- **Factory and passkey reset.** `ResetFlash` (`0x4000`) and `ResetPasskey`
  (`0x2000`) in `1402` wipe the motor's pairing. Without the key you have to
  pair again (read `1409` while the motor is unpaired or "pulled").
- **Under-voltage lockout.** When `UnderVoltageLockout` (`0x00200000`) is set,
  the motor refuses to move, from the wand too, until the battery recovers.

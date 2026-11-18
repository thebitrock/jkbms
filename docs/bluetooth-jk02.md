# Bluetooth (JK02)

The JK-PB BMS has a Bluetooth LE module. A central (phone, ESP32, PC) talks to
it with JK02 frames: 20-byte requests that start with `AA 55 90 EB` and 300-byte
responses that start with `55 AA EB 90`. The response bodies are the same data
blocks that Modbus exposes, so the [register map](register-map.md) is also the
JK02 field map.

✔ = checked on a device. ○ = expected behaviour, not yet checked on a device. ? = assumed meaning, not confirmed.

## 1. Discovery and GATT

| Item | Value | |
|---|---|---|
| Advertised name | `<Bluetooth name>-NN`. The name is the string at 0x1428; NN = pack address, 2 digits (`-01` for address 1). Several packs on one site are told apart by NN. | ✔ |
| Advertised services | 0xFFE0 and 0xFEE7 | ✔ |
| Data service | 0xFFE0 | ✔ |
| Data characteristic | 0xFFE1 (handle 0x000A on the test device): write without response + notify. Requests are written to it; responses arrive as notifications. | ✔ |
| Other characteristics | 0xFFE2 (write, notify) and 0xFFE3 (write) in the same service, and the services 0x1800, 0xFE59 and two vendor services. Not needed for JK02. | ✔ (seen) |
| PIN / pairing | The BMS does not check any password on the Bluetooth link. | ○ |
| Connections | One central at a time ? While another central was connected, a second one could connect but saw no services. | ? |

The module is a transparent bridge. MTU and the split of a 300-byte response into
notifications depend on the module, so always reassemble the frame yourself (see 4.3).
On the test device the BMS sent its own MTU exchange request (512) right after the
connection, and each 300-byte frame arrived as two notifications of 150 bytes. ✔

On a Linux host with BlueZ 5.87, the D-Bus GATT client connected and showed
"services resolved", but listed no characteristics. A plain ATT client on an L2CAP
socket (CID 4) worked. If your BlueZ client shows no characteristics, try that.

## 2. Session

1. Connect and enable notifications on 0xFFE1.
2. Write the **device-info request (0x97)**. The BMS answers with a type 0x03 frame.
3. Write the **settings request (0x96)**. The BMS answers with a type 0x01 frame and
   then a type 0x03 frame. ✔
4. After both requests, the BMS streams **type 0x02 frames** (telemetry)
   continuously, one frame every 0.4 s (about 2.5 per second). ✔

The BMS also sends the 20-byte acknowledgement (4.4, status 1) after 0x97 and after
0x96. ✔ Observed timing: 0x97 → type 0x03 after 0.13 s, then the ack; 0x96 → type
0x01 after 0.07 s, type 0x03 after 0.17 s, the ack, then the first type 0x02 after
0.6 s.
5. Writes are accepted only after step 3.

Before both 0x97 and 0x96 have been received, the BMS ignores every command except
0x66, 0x96, 0x97, 0x9B, 0x9D, 0xFE and 0xFF, without an answer. ○ Several of these
are dangerous (restart, factory reset, reboot, rename), so check every frame you
build.

When the central disconnects, the session is reset. After each reconnect, send 0x97
and 0x96 again. If a central connects but does not complete the handshake, the BMS
resets its Bluetooth module after a short time (about 20 s ?), and the link drops.

While a central is connected, the module sends `AT\r\n` (`41 54 0D 0A`) as separate
4-byte notifications, about every 135 ms, between the frames. ✔ Ignore any bytes
that are not part of a frame.

## 3. Request frame (central → BMS)

Exactly **20 bytes**. One request per write. Two requests joined into one write are
not 20 bytes and are ignored. ○

| Byte | Content |
|---|---|
| 0–3 | `AA 55 90 EB` |
| 4 | Command code (section 6) |
| 5 | Value length in bytes (0 for requests, 4 for numeric settings, string length for strings). Must be < 14. |
| 6–9 | Value, little-endian. Strings start at byte 6. |
| 10 | Cell index 0–31, only for code 0x23 (connection-wire resistance). Otherwise 0. |
| 11–18 | 0 |
| 19 | Checksum = (sum of bytes 0..18) & 0xFF |

Prepared frames:

```
0x97 device info   : AA 55 90 EB 97 00 00 00 00 00 00 00 00 00 00 00 00 00 00 11
0x96 settings      : AA 55 90 EB 96 00 00 00 00 00 00 00 00 00 00 00 00 00 00 10
0xC3 one telemetry : AA 55 90 EB C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 3D
```

Write examples:

```
Max balance current = 1000 mA (code 0x13):
                     AA 55 90 EB 13 04 E8 03 00 00 00 00 00 00 00 00 00 00 00 7C
Charging float mode on (code 0x30):
                     AA 55 90 EB 30 04 01 00 00 00 00 00 00 00 00 00 00 00 00 AF
Wire resistance of cell 5 = 200 µΩ (code 0x23, index in byte 10):
                     AA 55 90 EB 23 04 C8 00 00 00 05 00 00 00 00 00 00 00 00 6E
```

## 4. Response frames (BMS → central)

### 4.1 Data frame

**300 bytes.** ✔ (types 0x01, 0x02, 0x03, checksums valid)

| Byte | Content |
|---|---|
| 0–3 | `55 AA EB 90` |
| 4 | Frame type |
| 5 | Frame counter (counts up on each telemetry frame) |
| 6–297 | Body: 0x124 (292) bytes |
| 298 | 0 (0x02 in type 0x03 frames) |
| 299 | Checksum = (sum of bytes 0..298) & 0xFF |

| Type | Body | Sent after |
|---|---|---|
| 0x01 | Settings block, Modbus 0x1000–0x1123 | 0x96, a preset command |
| 0x02 | Telemetry block, Modbus 0x1200–0x1323 | streamed after the handshake; 0xC3 |
| 0x03 | Device-info block, Modbus 0x1400–0x1523 | 0x97, 0x96 |
| 0x05 | 256-byte log page | 0xA1 |
| 0x06 | Event records: start index in bytes 6–7, count in byte 8, then up to 12 entries of 24 bytes from byte 9. Frames repeat until the log is empty. | 0xA7 |

### 4.2 Field offsets

```
frame byte = 6 + (Modbus address − block base)
block base = 0x1000 (type 0x01), 0x1200 (type 0x02), 0x1400 (type 0x03)
```

All multi-byte values are **little-endian** (Modbus sends the same values
big-endian). For a U8+U8 field, the byte at the lower Modbus address is the first
byte in the frame. Units, ranges and meanings are in the
[register map](register-map.md).

Common telemetry fields (type 0x02):

| Frame byte | Modbus | Field | Type |
|---|---|---|---|
| 6 + 2·n | 0x1200 + 2·n | Cell *n* voltage | u16 mV |
| 70 | 0x1240 | Cell present bitmask | u32 |
| 74 / 76 | 0x1244 / 0x1246 | Average cell / max − min | u16 mV |
| 78 / 79 | 0x1248 / 0x1249 | Index of the highest / lowest cell | u8 |
| 80 + 2·n | 0x124A + 2·n | Balance-lead resistance | u16 mΩ |
| 144 | 0x128A | MOSFET temperature | s16 0.1 °C |
| 150 | 0x1290 | Pack voltage | u32 mV |
| 154 | 0x1294 | Pack power | u32 mW |
| 158 | 0x1298 | Pack current, + charge / − discharge | s32 mA |
| 162 / 164 | 0x129C / 0x129E | Temperature 1 / 2 | s16 0.1 °C |
| 166 | 0x12A0 | Alarm bitmask | u32 |
| 170 | 0x12A4 | Balance current | s16 mA |
| 172 / 173 | 0x12A6 / 0x12A7 | Balance state / **SOC %** | u8 / u8 |
| 174 | 0x12A8 | Remaining capacity | s32 mAh |
| 178 | 0x12AC | Full-charge capacity | u32 mAh |
| 182 | 0x12B0 | Cycle count | u32 |
| 186 | 0x12B4 | Total cycle capacity | u32 mAh |
| 190 / 191 | 0x12B8 / 0x12B9 | SOH % / pre-charge active | u8 / u8 |
| 194 | 0x12BC | Run time | u32 s |
| 198 / 199 | 0x12C0 / 0x12C1 | Charge MOS on / discharge MOS on | u8 / u8 |
| 214 / 215 | 0x12D0 / 0x12D1 | Temperature-sensor present bits / heating active | u8 / u8 |
| 218 | 0x12D4 | Emergency time left | u16 s |
| 236 | 0x12E6 | Heater current | s16 mA |
| 254 / 256 / 258 | 0x12F8 / 0x12FA / 0x12FC | Temperature 3 / 4 / 5 | s16 0.1 °C |
| 262 | 0x1300 | Clock, seconds since 2020-01-01 | u32 |

Common settings fields (type 0x01): settings value *k* (k = 0…33) at byte
6 + 4·k, in the order of the register map (for example RCV = byte 38, RFV = byte 42,
nominal capacity = byte 130). Function switches 0x1114 = bytes 282–283 (u16 LE);
heating start/stop temperatures = bytes 284 / 285; smart-sleep delay = byte 286.

Device-info fields (type 0x03): model = byte 6, hardware version = byte 22,
firmware version = byte 30, run time = byte 38, power-on count = byte 42,
Bluetooth name = byte 46, RCV time = byte 270, emergency duration = byte 273. ✔ (model,
versions, run time and power-on count decoded; the values match Modbus)

### 4.3 Reassembly

1. Append every notification to a buffer.
2. Find `55 AA EB 90`. Drop bytes before it.
3. Wait until 300 bytes from the header are in the buffer.
4. Check the checksum at byte 299. If it is wrong, drop the header bytes and search
   again.
5. Some module firmware sends extra bytes after byte 299. Ignore them.

### 4.4 Write acknowledgement

After a write command the BMS sends a 20-byte frame with the **request** header: ✔
(status 1; statuses 0 and 2 ○)

```
AA 55 90 EB C8 01 <status> 00 … 00 <checksum>
```

| Status | Meaning | Modbus equivalent |
|---|---|---|
| 1 | Accepted and saved | normal reply |
| 0 | Value out of range or a rule broken | exception 03 |
| 2 | Command not supported | exception 02 |

Example (accepted): `AA 55 90 EB C8 01 01 00 00 00 00 00 00 00 00 00 00 00 00 44`

Each accepted command also gives a short beep on the BMS. ○

## 5. Writes and the Modbus map

A JK02 write and a Modbus write of the same field go through the same checks and
have the same effect: same range, same cross-field rules, same EEPROM save, same
side effects. The ranges were checked on a device through Modbus. ✔ A same-value
write of code 0x30 (charging float mode = 1) over Bluetooth returned status 1. ✔ So the JK02 code
for each field is listed next to it in the [register map](register-map.md).

Do not write over Bluetooth and Modbus at the same moment. One of the two EEPROM
saves can be lost. ○

## 6. Command codes

### 6.1 Requests

| Code | Action | Allowed before the handshake |
|---|---|---|
| 0x97 | Send device-info frame (type 0x03) | yes |
| 0x96 | Send settings frame (type 0x01), then device-info frame | yes |
| 0xC3 | Send one telemetry frame (type 0x02) | no |
| 0xA1 | Send log frame (type 0x05) | no |
| 0xA7 | Send event-record frames (type 0x06) | no |

### 6.2 Numeric settings (length 4, value u32 LE)

| Code | Field (Modbus) | Code | Field (Modbus) |
|---|---|---|---|
| 0x01 | Smart-sleep voltage (0x1000) | 0x13 | Max balance current (0x1048) |
| 0x02 | Cell UVP (0x1004) | 0x14 | Charge over-temperature (0x104C) |
| 0x03 | Cell UVP recovery (0x1008) | 0x15 | Charge over-temperature recovery (0x1050) |
| 0x04 | Cell OVP (0x100C) | 0x16 | Discharge over-temperature (0x1054) |
| 0x05 | Cell OVP recovery (0x1010) | 0x17 | Discharge over-temperature recovery (0x1058) |
| 0x06 | Balance trigger delta (0x1014) | 0x18 | Charge under-temperature (0x105C) |
| 0x07 | SOC 100 % voltage (0x1018) | 0x19 | Charge under-temperature recovery (0x1060) |
| 0x08 | SOC 0 % voltage (0x101C) | 0x1A | MOSFET over-temperature (0x1064) |
| 0x09 | RCV (0x1020) | 0x1B | MOSFET over-temperature recovery (0x1068) |
| 0x0A | RFV (0x1024) | 0x1C | Cell count (0x106C) |
| 0x0B | Power-off voltage (0x1028) | 0x1D | Charge MOS enable (0x1070) |
| 0x0C | Charge over-current (0x102C) | 0x1E | Discharge MOS enable (0x1074) — 0 cuts the load |
| 0x0D | Charge OCP delay (0x1030) | 0x1F | Balancer enable (0x1078) |
| 0x0E | Charge OCP recovery (0x1034) | 0x20 | Nominal capacity (0x107C) — **resets SOC** |
| 0x0F | Discharge over-current (0x1038) | 0x21 | Short-circuit delay (0x1080) |
| 0x10 | Discharge OCP delay (0x103C) | 0x22 | Balance start voltage (0x1084) |
| 0x11 | Discharge OCP recovery (0x1040) | 0x23 | Wire resistance, cell index in byte 10 (0x1088 + 4·n) |
| 0x12 | Short-circuit recovery (0x1044) | 0x25 | Pre-charge time (0x110C) |

Code 0x24 (pack address) and 0x26 are not supported (status 2).

### 6.3 Switches (length 4, value 0 or 1)

Each code sets or clears one bit of 0x1114.

| Code | Bit | Function |
|---|---|---|
| 0x27 | 0 | Heating |
| 0x28 | 1 | Disable battery temperature sensors |
| 0x2B | 4 | Display always on |
| 0x2D | 6 | Smart sleep |
| 0x2E | 7 | Disable parallel current-limit module |
| 0x2F | 8 | Timed data logging |
| 0x30 | 9 | Charging float mode |
| 0x32 | 11 | Dry-contact alarm intermittent |
| 0x33 | 12 | Discharge OCP 2 |

Codes 0x31, 0x34, 0x35 and 0x36 are not supported on V19.x (status 2).

### 6.4 Byte settings

| Code | Field | Range |
|---|---|---|
| 0x37 / 0x38 | Heating start / stop temperature (0x1116 / 0x1117) | −40…100 °C, start < stop |
| 0x39 | Smart-sleep delay (0x1118) | 1–100 h |
| 0x3A / 0x3B | Discharge low-temperature protection / recovery (0x1122 / 0x1123) | −40…100 °C, protection < recovery |
| 0xB3 | RCV time (0x1504) | 1–250 × 0.1 h |
| 0xB5 | Emergency start duration (0x1507) | 1–30 min |
| 0xB6 | Protocol of the third serial port (0x1508) | < 32, enabled in its bitmap |
| 0xB7 | 0x1510 | 1–50 |
| 0xA9 / 0xAA / 0xAB | Source for buzzer / dry contact 1 / dry contact 2 (0x14E4 / 0x14E5 / 0x14E6) | < 15 |
| 0xAC … 0xB1 | Buzzer, dry contact 1, dry contact 2 trigger / release values (0x14E8 … 0x14FC) | no range check |
| 0xB2 | Data logging period (0x1500) | ≥ 5 s |

Code 0xB4 (RFV time) is not supported on V19.x (status 2).

### 6.5 Strings (value = ASCII from byte 6, length ≤ 12)

| Code | Field |
|---|---|
| 0x9B | Bluetooth name (0x1428). The module is renamed at once. Allowed before the handshake. |
| 0x9C | Passcode (0x1438) |
| 0x9E | Manufacturing date (0x1448), only while it is `000000` |
| 0x9F | Device name (0x1460) |
| 0xA0 | Setup passcode (0x1470) |
| 0xA2 | User data (0x1480) |

### 6.6 Commands

| Code | Command | Allowed before the handshake | Risk |
|---|---|---|---|
| 0x64 | Voltage calibration (value = real pack mV) | no | Wrong voltages |
| **0x65** | **Shutdown** | no | Load loses power |
| **0x66** | **Restart** | **yes** | Short loss of BMS logic |
| 0x67 | Current calibration (value = real mA; 0 at zero current) | no | Wrong current and SOC |
| 0x68 / 0x69 / 0x6A | Load Li-ion / LiFePO4 / LTO preset | no | Overwrites all thresholds, resets SOC |
| 0x6B | Emergency start toggle | no | Bypasses protection |
| 0x6C | Set the clock (seconds since 2020-01-01) | no | Low |
| 0x6E | SOC calibration ?, 1–99 % | no | Changes SOC |
| 0x6F | SOH calibration ?, 50–100 % | no | Changes SOH |
| **0x9D** | **Factory reset** | **yes** | High |
| 0xA3 | Full factory reset including calibration | no | Very high |
| **0xFF** | **Reboot** (into update mode ?) | **yes** | Do not use |
| 0x6D, 0x70, 0xC4, 0xFE | Service functions | — | Do not use |

None of the commands in 6.6 was tested on a device. ○

## 7. Bulk configuration frames

The vendor app imports a complete configuration with 300-byte request frames
(header `AA 55 90 EB`, byte 4 = 0x07 then 0x04, sent within a short time window).
This replaces the whole settings block and resets SOC. Client software should write
single fields instead.

## 8. Practical notes

- Read with notifications only. There is no need to poll after the handshake.
- Telemetry frames arrive several times per second. Update your display at a lower
  rate.
- For several packs, open one connection per pack and identify each pack by the
  `-NN` suffix of its name.
- Keep Bluetooth and RS485 Modbus access read-only at the same time, or write from
  one side only.

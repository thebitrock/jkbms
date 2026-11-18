# jkbms

Developer documentation for the communication protocols of **JK-BMS PB-series**
protection boards (JK-PB2A16S20P and related models). It explains how to read
data from the BMS and how to write settings and send commands.

The documents apply to firmware V19.x. Behaviour marked ✔ was checked on a
JK-PB2A16S20P (hardware 19A, firmware V19.30). Behaviour marked ○ is documented
but was not yet checked on a device.

Not affiliated with Jikong (JK-BMS). Use at your own risk: this device protects
a large battery.

## Interfaces

| Interface | Protocol | Document |
|---|---|---|
| RS485 | Modbus RTU slave, 115200 8N1 ✔ | [Modbus RTU](docs/modbus-rtu.md) |
| Bluetooth LE | JK02 frames (`AA 55 90 EB` requests, `55 AA EB 90` responses) | [Bluetooth (JK02)](docs/bluetooth-jk02.md) |
| CAN, RS232/RS485 to an inverter | Selectable inverter protocols | [Serial ports and CAN](docs/serial-ports-and-can.md) |

All interfaces read and write the same three data blocks:

| Block | Modbus addresses | JK02 frame type | Content |
|---|---|---|---|
| Settings | 0x1000–0x1123 | 0x01 | Protection thresholds, limits, switches (stored in EEPROM) |
| Telemetry | 0x1200–0x1323 | 0x02 | Cell voltages, current, temperatures, SOC, MOS state, alarms |
| Device info | 0x1400–0x1523 | 0x03 | Model, versions, counters, names, port protocols, dry contacts |
| Commands | 0x1600–0x1637 (write only) | command codes | Calibration, shutdown, presets, RTC, frame requests |

A field has the same offset in every block on every interface. Modbus sends it
big-endian; JK02 frames send it little-endian.

## Documents

- [Modbus RTU](docs/modbus-rtu.md): framing, addressing, data types, how to read, how to
  write, exceptions, timing, examples with real frames.
- [Register map](docs/register-map.md): every address with its meaning, unit, accepted
  range, cross-field rules and live example values.
- [Bluetooth (JK02)](docs/bluetooth-jk02.md): GATT, handshake, request and response frames,
  all command codes, write acknowledgement.
- [Serial ports and CAN](docs/serial-ports-and-can.md): ports, protocol numbers, baud
  rates, CAN bit rates and frame IDs, pack address and parallel packs.
- [Device behaviour](docs/device-behaviour.md): protection logic, SOC estimation, what a
  write does, presets, emergency start, EEPROM wear.

## Quick start (RS485)

```
Request  (read pack voltage 0x1290, 2 registers): 01 03 12 90 00 02 C1 5E
Response (53280 mV):                              01 03 04 00 00 D0 20 A6 2B
```

- 115200 baud, 8N1, slave address = the pack address (1 on a single pack).
- Addresses are byte offsets: one 16-bit register moves the address by 2.
- Function codes: 0x03 read, 0x10 write. Nothing else.

## Before you write anything

- Every accepted write is saved to EEPROM at once. Do not write on a timer or on
  every start-up.
- **Writing the battery capacity (0x107C) or loading a chemistry preset resets SOC**
  to a rough estimate from the cell voltage. The reset was seen on a device after a
  capacity write. ✔
- Some commands cannot be undone: shutdown, factory reset, calibration. See the
  [register map](docs/register-map.md#5-commands-0x1600-write-only).
- There is no authentication. Any device on the bus or any Bluetooth central can
  read the passcodes and change the protection settings.

## License

[MIT](LICENSE)

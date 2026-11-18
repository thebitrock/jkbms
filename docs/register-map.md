# Register map

Every address of the JK-PB BMS data blocks. The addresses are the Modbus RTU
addresses (byte offsets, see [Modbus RTU](modbus-rtu.md#2-addressing)). The same
fields appear in the Bluetooth JK02 frames at
`frame byte = 6 + (address − block base)`, little-endian (see
[Bluetooth](bluetooth-jk02.md)).

Columns:

- **Range / rules**: what the BMS accepts on a write. A value outside the range
  or a broken rule gives Modbus exception 03 (JK02 status 0).
- **Default**: factory value of the LiFePO4 preset.
- **Example**: value read from a working 16S LiFePO4 100 Ah pack.
- **✔** checked on a device. **○** expected, not yet checked on a device. **?** assumed
  meaning, not confirmed.
- In the **W** column, ✔ means that a write was accepted, read back and restored on
  a device. The ranges and cross-field rules are ○ unless a note says otherwise.
- **JK02**: the Bluetooth command code that writes the same field.

Write column: **W** = writable, **R** = read only, **—** = cannot be written
(Modbus exception 02, JK02 status 2).

---

## 1. Overview

| Block | Addresses | Access | Stored |
|---|---|---|---|
| Settings | 0x1000–0x1123 | R/W | EEPROM |
| Telemetry | 0x1200–0x1323 | R | RAM |
| Device info and extended settings | 0x1400–0x1523 | R, some W | EEPROM |
| Commands | 0x1600–0x1637 | W only | — |

---

## 2. Settings block (0x1000–0x1123)

All fields in 2.1–2.5 are 32-bit (2 registers). Signed fields are marked *s32*.

### 2.1 Cell voltage thresholds

`V` = any cell voltage from 1200 to 4400 mV.

| Addr | Name | Unit | Range / rules | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|---|
| 0x1000 | Smart-sleep voltage | mV | V | 3500 | 3500 | W ✔ | 0x01 |
| 0x1004 | **Cell UVP** (under-voltage protection) | mV | V; > 0x1028; < 0x101C | 2500 | 2800 | W ✔ | 0x02 |
| 0x1008 | Cell UVP recovery | mV | V; > 0x101C; < 0x1010 | 2900 | 3100 | W ✔ | 0x03 |
| 0x100C | **Cell OVP** (over-voltage protection) | mV | V; > 0x1020 | 3650 | 3600 | W ✔ | 0x04 |
| 0x1010 | Cell OVP recovery | mV | V; > 0x1008; < 0x1018 | 3400 | 3400 | W ✔ | 0x05 |
| 0x1014 | Balance trigger delta | mV | 3–998 | 10 | 10 | W ✔ (also over Bluetooth) | 0x06 |
| 0x1018 | SOC 100 % voltage | mV | V; > 0x1010; < 0x1020 | 3445 | 3445 | W ✔ | 0x07 |
| 0x101C | SOC 0 % voltage | mV | V; > 0x1004; < 0x1008 | 2700 | 2900 | W ✔ | 0x08 |
| 0x1020 | **RCV** (requested charge voltage per cell) | mV | V; > 0x1018; < 0x100C | 3450 | 3450 | W ✔ | 0x09 |
| 0x1024 | **RFV** (float voltage per cell) | mV | V | 3350 | 3350 | W ✔ | 0x0A |
| 0x1028 | System power-off voltage | mV | V; < 0x1004 | 2490 | 2600 | W ✔ | 0x0B |
| 0x1084 | Balance start voltage | mV | V | 3000 | 3400 | W ✔ | 0x22 |

The rules form one chain. Every write must keep it true: ○

```
power-off < UVP < SOC0% < UVP recovery < OVP recovery < SOC100% < RCV < OVP
 0x1028    0x1004  0x101C     0x1008         0x1010        0x1018   0x1020 0x100C
```

To raise a group of thresholds, write from the top of the chain (OVP first). To
lower them, write from the bottom (power-off first).

What the thresholds do:

- **UVP**: if any cell is below it, the discharge MOS turns off. It turns on again
  when all cells are above the UVP recovery voltage.
- **OVP**: if any cell is above it, the charge MOS turns off. It turns on again when
  all cells are below the OVP recovery voltage.
- **Power-off**: if a cell is below it, the board turns itself off to protect the
  cells.
- **RCV / RFV**: the charge voltage per cell that the BMS asks the inverter for (×
  cell count). RCV is used first. After the RCV time (0x1504) it changes to RFV, if
  charging float mode (0x1114 bit 9) is on.
- **SOC 100 %**: when a cell reaches it during charge, the remaining capacity is set
  to full.
- **Balancing** runs when the cells are above the balance start voltage and
  (max cell − min cell) > balance trigger delta.

### 2.2 Current limits and delays

| Addr | Name | Unit | Range | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|---|
| 0x102C | Charge over-current limit | mA | 1000–200000 | 200000 | 80000 | W ✔ | 0x0C |
| 0x1030 | Charge OCP delay | s | 2–600 | 3 | 3 | W ✔ | 0x0D |
| 0x1034 | Charge OCP recovery time | s | 2–600 | 60 | 60 | W ✔ | 0x0E |
| 0x1038 | Discharge over-current limit | mA | 1000–200000 | 200000 | 100000 | W ✔ | 0x0F |
| 0x103C | Discharge OCP delay | s | 10–600 | 300 | 30 | W ✔ | 0x10 |
| 0x1040 | Discharge OCP recovery time | s | 2–600 | 60 | 60 | W ✔ | 0x11 |
| 0x1044 | Short-circuit recovery time | s | 2–600 | 5 | 5 | W ✔ | 0x12 |
| 0x1048 | Maximum balance current | mA | **300–2000** | 2000 | 2000 | W ✔ (2001 → exception 03) | 0x13 |
| 0x1080 | Short-circuit detection delay | µs | 0–50 | 5 | 5 | W ✔ | 0x21 |

Over-current: if the current stays above the limit for the delay time, the MOS for
that direction turns off. It turns on again by itself after the recovery time.

### 2.3 Temperature limits (s32, 0.1 °C)

| Addr | Name | Range | Rules | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|---|
| 0x104C | Charge over-temperature | 300–1000 | > 0x1050 | 700 (70.0 °C) | 500 | W ✔ | 0x14 |
| 0x1050 | Charge over-temperature recovery | 300–1000 | < 0x104C; > 0x1060 | 600 | 450 | W ✔ | 0x15 |
| 0x1054 | Discharge over-temperature | 300–1000 | > 0x1058 | 700 | 550 | W ✔ | 0x16 |
| 0x1058 | Discharge over-temperature recovery | 300–1000 | < 0x1054 | 600 | 500 | W ✔ | 0x17 |
| 0x105C | **Charge under-temperature** | −450–200 | < 0x1060 | −100 (−10.0 °C) | 20 (2.0 °C) | W ✔ | 0x18 |
| 0x1060 | Charge under-temperature recovery | −450–200 | > 0x105C; < 0x1050 | 0 | 50 | W ✔ | 0x19 |
| 0x1064 | MOSFET over-temperature | 500–1100 | > 0x1068 | 800 | 800 | W ✔ | 0x1A |
| 0x1068 | MOSFET over-temperature recovery | 500–1100 | < 0x1064 | 700 | 700 | W ✔ | 0x1B |

Charge under-temperature blocks charging below the limit. LiFePO4 must not be
charged below 0 °C, so keep this limit at 0 °C or above for LiFePO4 without a
heater.

### 2.4 Pack configuration and switches

| Addr | Name | Values | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|
| 0x106C | Cell count | 2–16 (on 16S boards) | 16 | 16 | W ✔ (same value) | 0x1C |
| 0x1070 | Charge MOS enable | 0 = off, 1 = on | 1 | 1 | W ✔ (0 turned the charge MOS off at once, 1 back on) | 0x1D |
| 0x1074 | Discharge MOS enable | 0 = off, 1 = on. **0 cuts power to the load.** | 1 | 1 | W ✔ (0 turned the discharge MOS off at once; after 1 it came back within a few seconds) | 0x1E |
| 0x1078 | Balancer enable | 0 = off, 1 = on | 1 | 1 | W ✔ (1→0→1) | 0x1F |
| 0x107C | **Nominal capacity** | 2000–2000000 mAh. **A write resets SOC** ([details](device-behaviour.md#22-soc-re-estimate)). | 200000 | 100000 | W ✔ | 0x20 |

A wrong cell count raises the "cell quantity" alarm.

### 2.5 Connection-wire resistance

| Addr | Name | Unit | Range | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|---|
| 0x1088 + 4·n (n = 0…31) | Resistance of the bus bar or cable between cell *n* and cell *n*+1 | µΩ | 0–2000000 | 0 | 0 | W ✔ (all 32) | 0x23, cell index in byte 10 |

The BMS subtracts I × R from the measured cell voltage under current. 0 = no
compensation.

### 2.6 Board configuration (0x1108–0x1123)

| Addr | Type | Name | Values | Default | Example | W | JK02 |
|---|---|---|---|---|---|---|---|
| 0x1108 | u32 | Pack address (DevAddr) | Set by DIP switches. 0 = parallel master ? (see [Serial ports](serial-ports-and-can.md#4-pack-address-and-parallel-packs)). | 1 | 1 | — ✔ | — |
| 0x110C | u32 | Discharge pre-charge time | 0–300 s | 5 | 5 | W ✔ | 0x25 |
| 0x1110 | u32 | Not documented (factory constant ?) | — | 500000 | 500000 | — ○ | — |
| 0x1114 | u16 | Function switches | bitfield below | 0x1010 | 0x1200 | W ✔ (bits 4, 6) | 0xBE (all) / per bit |
| 0x1116 | s8 pair | Heating start temperature (high byte), heating stop temperature (low byte) | −40…100 °C each; start < stop | −10 / 0 | −10 / 0 | W ✔ (same value) | 0xBF (both), 0x37 / 0x38 |
| 0x1118 | U8+U8 | Smart-sleep delay (high byte) | 1–100 h. A write also restarts the sleep countdown 0x1308. ✔ | 24 | 24 | W ✔ (same value) | 0x39 |
| 0x1120 | — | Reserved | — | — | — | — | — |
| 0x1122 | s8 pair | Discharge low-temperature protection (high byte) / recovery (low byte) ? (−20 / −15 °C on the test device; same-value write accepted ✔) | −40…100 °C each; protection < recovery | −35 / −30 | −20 / −15 | W ○ | 0xC6 (both), 0x3A / 0x3B |

The heating temperatures act only when heating is enabled (0x1114 bit 0). The
example values were consistent with the smart-sleep countdown: 0x1118 = 24 h and
0x1308 = 86400 s. ✔

**0x1114 function switches** (bit 0 = least significant bit of the u16):

| Bit | Mask | Function when 1 | Writable | JK02 | Default | Example |
|---|---|---|---|---|---|---|
| 0 | 0x0001 | Heating enabled | yes | 0x27 | 0 | 0 |
| 1 | 0x0002 | Battery temperature sensors disabled | yes | 0x28 | 0 | 0 |
| 2 | 0x0004 | GPS heartbeat check | **no** | — | 0 | 0 |
| 3 | 0x0008 | Shared port is RS485 (0 = CAN) | **no** | — | 0 | 0 |
| 4 | 0x0010 | Display always on | yes ✔ | 0x2B | **1** | 0 |
| 5 | 0x0020 | Special charger detection | **no** | — | 0 | 0 |
| 6 | 0x0040 | Smart sleep | yes | 0x2D | 0 | 0 |
| 7 | 0x0080 | Parallel current-limit module disabled. Setting it clears the related fault. | yes | 0x2E | 0 | 0 |
| 8 | 0x0100 | Timed data logging | yes | 0x2F | 0 | 0 |
| 9 | 0x0200 | **Charging float mode** (RCV → RFV after the RCV time) | yes | 0x30 | 0 | **1** |
| 10 | 0x0400 | Emergency-button trigger | **no** | — | 0 | 0 |
| 11 | 0x0800 | Dry-contact alarm intermittent | yes | 0x32 | 0 | 0 |
| 12 | 0x1000 | Discharge OCP 2 (second over-current level) | yes | 0x33 | **1** | **1** |
| 13 | 0x2000 | Discharge OCP 3 | **no** | — | 0 | 0 |

A write of 0x1114 changes only the writable bits (mask 0x1BD3). The other bits keep
their value. ○

---

## 3. Telemetry block (0x1200–0x1323, read only)

The examples were read from an idle 16S pack (current 0, cells ≈ 3.33 V). ✔ = the
value was decoded and checked against other readings.

| Addr | Type | Unit | Name | Example | |
|---|---|---|---|---|---|
| 0x1200 + 2·n (n = 0…31) | u16 | mV | Cell *n* voltage. Cells above the cell count read 0. | 3329–3331 | ✔ |
| 0x1240 | u32 | bitmask | Bit *n* = cell *n* present | 0x0000FFFF | ✔ |
| 0x1244 | u16 | mV | Average cell voltage | 3330 | ✔ |
| 0x1246 | u16 | mV | Max cell − min cell | 2 | ✔ |
| 0x1248 | U8+U8 | index | High byte: index (0-based) of the highest cell. Low byte: index of the lowest cell. | 0x0F03 | ✔ |
| 0x124A + 2·n (n = 0…31) | u16 | mΩ | Measured resistance of the balance lead of cell *n*. A high value means a bad crimp or connector. | 61–65 | ✔ |
| 0x128A | s16 | 0.1 °C | MOSFET (power board) temperature | 194 | ✔ |
| 0x128C | u32 | bitmask | Bit *n* = balance lead *n* resistance alarm | 0 | ✔ |
| 0x1290 | u32 | mV | Pack voltage | 53280 | ✔ |
| 0x1294 | u32 | mW | Pack power (absolute) | 0 | ✔ |
| 0x1298 | s32 | mA | **Pack current. + = charge, − = discharge.** | 0 | ✔ |
| 0x129C | s16 | 0.1 °C | Battery temperature 1 | 174 | ✔ |
| 0x129E | s16 | 0.1 °C | Battery temperature 2 | 175 | ✔ |
| 0x12A0 | u32 | bitmask | Alarms (bit list below) | 0 | ✔ |
| 0x12A4 | s16 | mA | Balance current | 0 | ✔ |
| 0x12A6 | U8+U8 | — / % | High byte: balance state (0 = off, 1 = charging cells, 2 = discharging cells). **Low byte: SOC %.** | 0x0064 | ✔ |
| 0x12A8 | s32 | mAh | Remaining capacity | 99915 | ✔ |
| 0x12AC | u32 | mAh | Full-charge capacity | 100000 | ✔ |
| 0x12B0 | u32 | — | Cycle count | 1 | ✔ |
| 0x12B4 | u32 | mAh | Total charged capacity over the lifetime | 110199 | ✔ |
| 0x12B8 | U8+U8 | % / flag | High byte: SOH %. Low byte: pre-charge active (1 = on). | 0x6400 | ✔ |
| 0x12BA | u16 | bitmask | User alarm word 1 (bit meanings not published by the vendor) | 0 | ✔ |
| 0x12BC | u32 | s | Run time since power-up | 395945 | ✔ |
| 0x12C0 | U8+U8 | flag | High byte: **charge MOS on**. Low byte: **discharge MOS on**. | 0x0101 | ✔ |
| 0x12C2 | u16 | bitmask | User alarm word 2 | 0 | ✔ |
| 0x12C4 | u16 | s | Time until discharge OCP releases | 0 | ✔ |
| 0x12C6 | u16 | s | Time until discharge short-circuit protection releases | 0 | ✔ |
| 0x12C8 | u16 | s | Time until charge OCP releases | 0 | ✔ |
| 0x12CA | u16 | s | Time until charge short-circuit protection releases | 0 | ✔ |
| 0x12CC | u16 | s | Time until UVP releases | 0 | ✔ |
| 0x12CE | u16 | s | Time until OVP releases | 0 | ✔ |
| 0x12D0 | U8+U8 | bitmask / flag | High byte: temperature-sensor present bits (1 = present). Low byte: heating active. ○ | 0xFF00 | ✔ |
| 0x12D2 | u16 | — | Reserved | 0x0100 | |
| 0x12D4 | u16 | s | Emergency start time left. 0 = not active. | 0 | ✔ |
| 0x12D6 | u16 | — | Discharge current correction factor | 1 | ✔ |
| 0x12D8 | u16 | mV | Charge current sensor voltage (zero offset) | 0 | ✔ |
| 0x12DA | u16 | mV | Discharge current sensor voltage (zero offset) | 1 | ✔ |
| 0x12DC | float32 | factor | Pack voltage calibration factor (set by command 0x1600, range 2.95–3.05) | 2.99487 | ✔ |
| 0x12E4 | u16 | 10 mV | Pack voltage, lower resolution | 5328 | ✔ |
| 0x12E6 | s16 | mA | Heater current | 0 | ✔ |
| 0x12F0 | u32 | 0.1 s | System tick counter | 419119 | ✔ |
| 0x12F8 | s16 | 0.1 °C | Battery temperature 3 | 194 | ✔ |
| 0x12FA | s16 | 0.1 °C | Battery temperature 4 | 174 | ✔ |
| 0x12FC | s16 | 0.1 °C | Battery temperature 5 | 178 | ✔ |
| 0x1300 | u32 | s | Real-time clock, seconds since 2020-01-01 00:00. Set with command 0x1612. | 217226343 | ✔ |
| 0x1304 | u32 | — | Not documented | 166 | |
| 0x1308 | u32 | s | Time until the BMS goes to sleep (= smart-sleep delay × 3600 after a write of 0x1118) | 86400 | ✔ |

**0x12A0 alarm bits:** ○ (all 0 on the test pack)

| Bit | Alarm | Bit | Alarm |
|---|---|---|---|
| 0 | Balance wire resistance | 11 | Cell UVP |
| 1 | MOSFET over-temperature | 12 | Battery UVP |
| 2 | Cell quantity mismatch | 13 | Discharge OCP |
| 3 | Current sensor error | 14 | Discharge short circuit |
| 4 | Cell OVP | 15 | Discharge over-temperature |
| 5 | Battery OVP | 16 | Charge MOS fault |
| 6 | Charge OCP | 17 | Discharge MOS fault |
| 7 | Charge short circuit | 18 | GPS disconnected |
| 8 | Charge over-temperature | 19 | Change the password in time |
| 9 | Charge under-temperature | 20 | Discharge on failed |
| 10 | Internal communication error | 21 | Battery over-temperature alarm |

---

## 4. Device info and extended settings (0x1400–0x1523)

**This block returns the passcodes in plain text to any reader.** Do not log or
publish dumps of 0x1400–0x14AF.

| Addr | Type | Name | Values / rules | Example | W | JK02 |
|---|---|---|---|---|---|---|
| 0x1400 | ASCII 16 | Model | | `JK-PB2A16S20P` | R ✔ | |
| 0x1410 | ASCII 8 | Hardware version | | `19A` | R ✔ | |
| 0x1418 | ASCII 8 | Firmware version | | `19.30` | R ✔ | |
| 0x1420 | u32 | Total run time | s | 395700 | R ✔ | |
| 0x1424 | u32 | Power-on count | | 60 | R ✔ | |
| 0x1428 | ASCII 16 | **Bluetooth name.** Advertised as `<name>-NN`, NN = pack address (2 digits). A write renames the BLE module. | ≤ 12 characters on write (6 registers). The factory name can be longer (15 characters on the test device), so a same-value write is not possible: it would cut the name. ✔ | (16 chars) | W ○ | 0x9B |
| 0x1438 | ASCII 16 | Passcode | ≤ 12 characters | — | W ○ | 0x9C |
| 0x1448 | ASCII 8 | Manufacturing date `YYMMDD` | Write-once: only while the stored value is `000000` | `260925` | W ○ | 0x9E |
| 0x1450 | ASCII 16 | Factory identifier (same text as the factory Bluetooth name) | | — | R ✔ | |
| 0x1460 | ASCII 16 | Device name | ≤ 12 characters. Write exactly 6 registers (12 bytes); 1, 2, 4, 7 or 8 registers → exception 03 ✔ | `JK-BMS` | W ✔ (same value) | 0x9F |
| 0x1470 | ASCII 16 | Setup passcode | ≤ 12 characters | — | W ○ | 0xA0 |
| 0x1480 | ASCII 16 | User data | ≤ 12 characters. Write exactly 6 registers ✔ | `JK-BMS` | W ✔ (same value) | 0xA2 |
| 0x1490 | ASCII 32 | Reserved string | | | R | |
| 0x14B0 | u16 | Not documented | no range check | 0x0000 | W ✔ (same value) | 0xA4 |
| 0x14B2 | U8+U8 | High byte: protocol number of the first serial port (on the test device **not** the RS485 connector used for Modbus, see [Serial ports](serial-ports-and-can.md)). Low byte: CAN protocol number. | Each number < 128 and enabled in its bitmap below | 0x0100 | W ✔ (0, 4 and back to 1 over Bluetooth) | 0xC0 (value byte 6 = high byte, byte 7 = low byte ✔) |
| 0x14B4 | bitmap | Protocols that may be selected for the port set by 0x14B2 high byte (bit *n* = protocol *n*) | | `FB E6` = 0, 1, 3–7, 9, 10, 13–15 | R ✔ | |
| 0x14C4 | 16 bytes | Bitmap of protocols that may be selected for CAN | | `FF 0F …` (0–11) | R ✔ | |
| 0x14D4 | U8 + bitmap | High byte: protocol number of the second serial port. From the low byte on: its enable bitmap. | | `01 FB E6` (protocol 1) | — ✔ | |
| 0x14E4 | U8+U8 | High byte: alarm source of the display buzzer. Low byte: source of dry contact 1. | each < 15 | 0x0908 | W ✔ (same value) | 0xC1, 0xA9 / 0xAA |
| 0x14E6 | U8+U8 | High byte: source of dry contact 2. Low byte: read only. | < 15 | 0x0001 | W ✔ (same value) | 0xAB |
| 0x14E8 | s32 | Buzzer trigger value | no range check | 100 | W ✔ | 0xAC |
| 0x14EC | s32 | Buzzer release value | no range check | 95 | W ✔ | 0xAD |
| 0x14F0 | s32 | Dry contact 1 trigger value | no range check | 60 | W ✔ | 0xAE |
| 0x14F4 | s32 | Dry contact 1 release value | no range check | 50 | W ✔ | 0xAF |
| 0x14F8 | s32 | Dry contact 2 trigger value | no range check | 0 | W ✔ | 0xB0 |
| 0x14FC | s32 | Dry contact 2 release value | no range check | 0 | W ✔ | 0xB1 |
| 0x1500 | s32 | Data logging period | 5–864000 s | 3600 | W ✔ | 0xB2 |
| 0x1504 | U8+U8 | High byte: **RCV time** (time at RCV before the change to RFV). Low byte: read only. | 1–250 × 0.1 h | 0x0532 (0.5 h) | W ✔ (high byte stored, low byte ignored) | 0xC2, 0xB3 |
| 0x1506 | U8+U8 | High byte: CAN protocol library version (read only, a changed high byte is not stored ✔). Low byte: emergency start duration. **A write also arms the emergency start** with the new duration ✔. | 1–30 min | 0x011E (30 min) | W ✔ (29, back to 30; then cancel with 0x1610) | 0xB5 |
| 0x1508 | U8 + 7 bytes | Protocol number of the third serial port, then its 7-byte enable bitmap | < 32 and enabled | | W ○ | 0xB6 |
| 0x1510 | U8+U8 | Not documented (pack-related count ?) | 1–50 | | W ○ | 0xC7 / 0xB7 |

Each dry contact and the buzzer switch on when the selected source crosses the
trigger value and switch off at the release value. The source numbers select the
watched quantity (for example SOC). The vendor does not publish the source list;
read the names in the vendor app.

---

## 5. Commands (0x1600, write only)

Commands are written with FC10 like settings. They cannot be read (exception 02).
Each command has a fixed size; send exactly that many registers.

| Addr | Size | Command | Value | Effect | Risk | JK02 |
|---|---|---|---|---|---|---|
| 0x1600 | u32 | Voltage calibration | Real pack voltage in mV, measured with a good meter | The BMS computes a new pack voltage factor (0x12DC). The new factor must be 2.95–3.05, or the command is rejected. | Wrong value = wrong voltages | 0x64 |
| 0x1604 | u16 | **Shutdown** | any | The board turns off after about 1 s. | Load loses power | 0x65 |
| 0x1606 | u32 | Current calibration | Real current in mA. Send 0 (or < 51) at zero current to set the zero offset. | Gain must stay within ±20 %. | Wrong value = wrong current and SOC | 0x67 |
| 0x160A | u16 | Load Li-ion (NMC) preset | any | Overwrites all thresholds, **resets SOC** | High | 0x68 |
| 0x160C | u16 | Load LiFePO4 preset | any | Overwrites all thresholds, **resets SOC** | High | 0x69 |
| 0x160E | u16 | Load LTO preset | any | Overwrites all thresholds, **resets SOC** | High | 0x6A |
| 0x1610 | u16 | Emergency start (toggle) ✔ | any | Turns the discharge MOS on for the emergency duration (0x1506 low byte) while a protection is active. A second write cancels it. ✔ (0x12D4 started at 30 min and went to 0 after the second write) | Bypasses protection | 0x6B |
| 0x1612 | u32 | Set the clock ✔ | Seconds since 2020-01-01 00:00 | Sets 0x1300 (checked with +1, +60, +600 s) | Low | 0x6C |
| 0x1616 | u16 | Restart ✔ | any | Replies, then restarts. The boot loader prints a short text banner on RS485 before the application starts. Modbus answered again within about 1 s; the power-on count went up by 1. Settings and SOC are kept. | Short power loss of the BMS logic | 0x66 |
| 0x1618 | u16 | **Factory reset** (keeps calibration) | any | All settings return to defaults | High | 0x9D |
| 0x161A | u16 | **Full factory reset** (also clears calibration) | any | Voltage and current accuracy is lost | **Very high** | 0xA3 |
| 0x161C | u16 | Send device-info frame (JK02 type 0x03) | any | Over Modbus: the 300-byte frame comes back on RS485, followed by the normal FC10 reply ✔ | None | 0x97 |
| 0x161E | u16 | Send settings frame (JK02 type 0x01) | any | Same as 0x161C ✔ | None | 0x96 |
| 0x1620 | u16 | Send telemetry frame (JK02 type 0x02) | any | Same as 0x161C ✔ | None | 0xC3 |
| 0x1622 | u16 | Send event-record frames (JK02 type 0x06) | any | Over Modbus: all frames come back on RS485 (15 × 300 bytes on the test device), then the FC10 reply ✔ | None | 0xA7 |
| 0x1624 | u16 | Send log frame (JK02 type 0x05) | any | Same as 0x161C ✔ | None | 0xA1 |
| 0x1626 | u16 | Reboot (into update mode ?) | any | | **Do not use** | 0xFF |
| 0x1628, 0x162A, 0x162C, 0x162E | — | Service and factory functions | — | — | **Do not use** | 0xC4, 0xFE, 0xC5, 0x6D |
| 0x1630 | u16 | SOC calibration ? | 1–99 % | **Not supported on V19.30:** exception 02 over Modbus, status 2 over Bluetooth ✔ | — | 0x6E |
| 0x1632 | u16 | SOH calibration ? | 50–100 % | **Not supported on V19.30:** exception 02 over Modbus, status 2 over Bluetooth ✔ | — | 0x6F |
| 0x1634 | u32 | Not documented | ≤ 20000000 | | Do not use | 0x70 |

Checked on a device ✔: 0x1610, 0x1612, 0x1616, 0x161C, 0x161E, 0x1620, 0x1622, 0x1624,
and 0x1630 / 0x1632 (not supported). A read of 0x1600 gives exception 02 ✔. The other
commands were not tested ○.

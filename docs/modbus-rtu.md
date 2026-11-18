# Modbus RTU

The RS485 port of the JK-PB BMS is a Modbus RTU slave. This document explains
the transport, the addressing model, the data types, and the exact rules for
reads and writes. The meaning of each address is in the
[register map](register-map.md).

✔ = checked on a device. ○ = expected behaviour, not yet checked on a device. ? = assumed meaning, not confirmed. Device: JK-PB2A16S20P, V19.30.

## 1. Transport

| Item | Value | |
|---|---|---|
| Physical layer | RS485, 2-wire (A/B). If A and B are swapped, the BMS does not reply at all. | ✔ |
| Line format | **115200 baud, 8 data bits, no parity, 1 stop bit** | ✔ |
| Slave address | The pack address (DIP switches). 1 on a single pack. It cannot be changed over Modbus. | ✔ |
| Function codes | **0x03** Read Holding Registers, **0x10** Write Multiple Registers | ✔ |
| Other function codes | 0x01, 0x04, 0x06 → exception **01** ✔. Others → exception 01 ○ | |
| Broadcast (address 0) | Not answered | ✔ |
| Other addresses | Not answered | ✔ |
| CRC | CRC-16/MODBUS (polynomial 0x8005 reflected = 0xA001, init 0xFFFF), low byte first | ✔ |
| Bad CRC | The BMS replies with exception **04** (`01 83 04 40 F3`), with a valid CRC | ✔ |

The Modbus slave runs on a serial port whose protocol number is 1. Do not change
the protocol numbers in 0x14B2 or 0x1508 from a Modbus connection: if you change
the number of the port you use, the Modbus slave stops answering on it. See
[Serial ports](serial-ports-and-can.md).

## 2. Addressing

**Addresses are byte offsets.** Each data block is a byte array, and the
Modbus address is `block base + byte offset`. One 16-bit register therefore
covers 2 address units, and a 32-bit value covers 4. ✔

```
address 0x1200  -> cell 0 voltage (u16)
address 0x1202  -> cell 1 voltage (u16)
address 0x1290  -> pack voltage (u32, uses 0x1290..0x1293)
```

A read of N registers at address A returns the 16-bit words at
A, A+2, A+4, …, A+2·(N−1). ✔ Most Modbus libraries count registers, not bytes:
pass the address as it is, and divide the data by 2 bytes per register as usual.
The only difference is that the next value starts at address + 2, not + 1.

### Readable windows ✔

| Block | Readable addresses | Size |
|---|---|---|
| Settings | 0x1000–0x1123 | 0x124 bytes |
| Telemetry | 0x1200–0x1323 | 0x124 bytes |
| Device info | 0x1400–0x1523 | 0x124 bytes |
| Commands | 0x1600… | **not readable** (exception 02) |

### Read rules ✔

| Rule | If broken |
|---|---|
| The start address is even | exception 02 |
| Quantity ≤ 123 registers | exception 02 |
| The whole request stays inside one window | exception 02 |
| The start address is inside a window | exception 02 |

You can read a complete block in two requests, for example 0x1200 × 123 and
0x12F6 × 23.

## 3. Data types

| Type | Registers | Encoding on the wire |
|---|---|---|
| u16 / s16 | 1 | Big-endian. s16 is two's complement. |
| u32 / s32 | 2 | Big-endian, **high word first**. Example: `00 00 D0 20` = 53280. |
| U8+U8 | 1 | Two independent bytes. **The byte at the lower address is the high byte of the register.** Example: 0x12A6 = `00 64` → byte 0x12A6 = 0 (balance state), byte 0x12A7 = 100 (SOC). |
| s8 pair | 1 | Two signed bytes, for temperature pairs (°C). Example: 0x1116 = `F6 00` → −10 °C, 0 °C. |
| float32 | 2 | IEEE 754, big-endian. Only 0x12DC. |
| ASCII | 4–16 | Zero-padded string, first character at the lowest address. |
| Bitmask | 1–2 | Bit 0 = least significant bit of the value. |

Units are given per field in the [register map](register-map.md): mV, mA, mAh,
s, 0.1 °C, µΩ, mΩ.

## 4. Reading

### Example: read the pack voltage ✔

```
Request : 01 03 12 90 00 02 C1 5E
          |  |  |---| |---| |---|
          |  |  addr  qty=2  CRC
          |  FC03
          slave 1
Response: 01 03 04 00 00 D0 20 A6 2B      -> 0x0000D020 = 53280 mV
```

### Example: read the current (signed) ✔

```
Request : 01 03 12 98 00 02 40 9C
Response: 01 03 04 FF FF CF 2C AE 3A      -> 0xFFFFCF2C = -12500 mA (discharge)
```

Positive current is charge, negative current is discharge.

### Example: read 16 cell voltages ✔

```
Request : 01 03 12 00 00 10 41 7E         -> 32 data bytes, 16 × u16 mV
```

### Recommended polling

- Telemetry: read 0x1200–0x1323 in two requests. Once every 1–2 s is enough; the
  values change slowly.
- Settings and device info change only when someone writes them. Read them at
  start-up and after your own writes.
- Wait for each reply (or a timeout of about 300 ms) before you send the next
  request. Use a gap of at least 3.5 characters (≈ 0.3 ms at 115200) between frames.
- Now and then the BMS did not answer a request for a few seconds, also when
  nothing else was on the bus. Right after a start-up or a button press, writes got
  no reply and were not applied for about 10 s to 1 min, while reads worked. Retry after the timeout. Before you repeat a write,
  read the field back: a repeated write of 0x107C resets SOC even when the first one
  was applied.
- Only one master may poll the bus. Two masters on the same RS485 pair collide and
  produce CRC errors on both sides.

## 5. Writing

### Write rules

- Use **FC10** only. FC06 is not supported (exception 01). ✔
- **One field per request.** The start address must be the exact start of a
  writable field, and the quantity must be exactly the size of that field:
  2 registers for a u32, 1 register for a 16-bit or U8+U8 field. ✔ (u32, u16 fields)
  - Start address that is not a writable field → exception 02. ✔ (0x1002, the
    second word of a u32)
  - Wrong quantity for the field → exception 03. ✔ (1 register to a u32; 4 registers
    over two fields)
- Value outside the accepted range, or a cross-field rule broken → exception 03. ✔
  (CurBalanMax = 2001 rejected; SOC 100 % voltage above RCV rejected)
- Field that cannot be written over Modbus → exception 02. ✔ (0x1108 DevAddr,
  0x14D4)
- Every accepted write is applied at once and saved to EEPROM. There is no separate
  "save" command. ✔ (0x1014 changed, BMS restarted, value kept)
- After a write, read the field back. The reply to FC10 only echoes the address and
  the quantity.

### Byte fields in a 16-bit register

Some writable fields are one byte. When you write such a field, the **high byte** of
the register is the value and the low byte is ignored, unless the field is a byte
pair (both bytes are written). ✔ (0x1504: a high-byte change was stored, a low-byte
change was ignored)

| Address | Written bytes |
|---|---|
| 0x1114 | u16 bitfield, but only some bits can change (see the register map) |
| 0x1116, 0x1122 | both bytes (temperature pair) |
| 0x1118, 0x1504 | high byte only |
| 0x14B2, 0x14E4, 0x1506 | see the register map |

### Example: write VolCellOV = 3600 mV ✔

```
Request : 01 10 10 0C 00 02 04 00 00 0E 10 3B 96
          |  |  |---| |---| |  |---------| |---|
          |  |  addr  qty=2 |  value       CRC
          |  FC10           byte count = 4
Response: 01 10 10 0C 00 02 85 0B
```

### Example: write the heating start/stop temperatures (−10 °C / 0 °C) ✔

```
Request : 01 10 11 16 00 01 02 F6 00 E2 07
```

### Exceptions

| Code | Name | When |
|---|---|---|
| 01 | Illegal function | Any function code other than 0x03 and 0x10 ✔ |
| 02 | Illegal data address | Odd address, outside a window, too many registers, read of 0x1600, write to a field that cannot be written ✔ |
| 03 | Illegal data value | Value out of range, cross-field rule broken, wrong quantity for the field ✔ (range) |
| 04 | Slave device failure | Bad CRC ✔ |

Exception frame: `slave, FC | 0x80, code, CRC`. Example: `01 83 02 C0 F1`.

## 6. Pitfalls

- **SOC reset.** Writing CapBatCell (0x107C) or loading a chemistry preset makes the
  BMS replace the remaining capacity with an estimate from the average cell voltage.
  On a flat LiFePO4 curve this estimate can be far off. This happens even when you
write the same value. ✔ See
  [Device behaviour](device-behaviour.md#22-soc-re-estimate).
- **EEPROM wear.** Write only when a value really changes. Never write the same value
  on every start-up or on a schedule.
- **Cross-field rules.** Thresholds depend on each other (for example UVP < SOC-0 %
  voltage < UVP recovery). To move a group of thresholds, write them in an order
  that keeps every rule true after each single write. The order is in the
  [register map](register-map.md#21-cell-voltage-thresholds).
- **Bluetooth at the same time.** A write over Bluetooth and a write over Modbus at
  the same moment can lose one of the two EEPROM saves. Reads are not affected. ○
- **Commands are not values.** The 0x1600 block cannot be read back, and most commands
  cannot be undone.

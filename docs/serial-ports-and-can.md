# Serial ports and CAN

The JK-PB BMS has several communication ports. Each serial port and the CAN port
runs one protocol from a numbered list. The protocol number decides the baud rate
and the frame format on that port.

✔ = checked on a device. ○ = not yet checked on a device.

## 1. Ports and protocol selectors

| Port | Protocol number | Selector | Writable |
|---|---|---|---|
| RS485 (Modbus port) | 1 = Modbus RTU slave ✔ | 0x14B2 high byte | yes ✔ (same value) |
| CAN | see section 3 | 0x14B2 low byte | yes ○ |
| Second serial port | — | 0x14D4 | no ✔ |
| Third serial port | — | 0x1508 | yes ○ |
| Bluetooth module | JK02 frames, fixed | — | — |

A protocol number can be selected only if its bit is set in the enable bitmap for
that port. The vendor app builds its protocol drop-down lists from these bitmaps:

| Bitmap | Port | Example |
|---|---|---|
| 0x14B4 (16 bytes) | RS485 | |
| 0x14C4 (16 bytes) | CAN | `FF 0F` = protocols 0–11 ✔ |
| 0x1509 (7 bytes) | third serial port | |

Bit *n* of the bitmap (byte n/8, bit n%8) enables protocol *n*.

Which serial connector on the board belongs to which port can differ between board
revisions. Check it with the vendor manual for your board. ○

**Do not change the protocol of the port you are connected to.** If you change the
RS485 port from protocol 1 to another number, the Modbus slave stops answering on it,
and you must change it back from the display or the app.

## 2. Serial protocol list

All serial protocols use 8N1. ○

| No. | Baud | Notes |
|---|---|---|
| 0 | 115200 | |
| **1** | **115200** | **JK Modbus RTU slave** (see [Modbus RTU](modbus-rtu.md)) ✔ |
| 3 | 9600 | Also forced on the second serial port in service mode |
| 4 | 9600 | |
| 5 | 115200 | |
| 6 | 9600 | |
| 7 | 9600 | |
| 9 | 9600 | |
| 10 | 115200 | |
| 13, 14 | 9600 | |
| 15 | 115200 | |
| 21 | 115200 | Used on the second serial port when the pack address is 0 (parallel master) |
| 2, 8, 11, 12, 16–20 | 9600 | Empty slots |

The inverter brand names for these numbers are shown only in the vendor app. Read
the name for a number there.

## 3. CAN

### 3.1 CAN protocol list

Each CAN protocol sets its own bit rate. Identifiers are 11-bit unless noted. ○

| No. | Bit rate | Identifiers used | Notes |
|---|---|---|---|
| 0 | 250 kbit/s | 0x2F4, 0x4F4, 0x5F4, 0x7F4, 0x108, 0x10E | Service-mode default |
| 1 | 500 kbit/s | 29-bit identifiers | |
| 2 | 500 kbit/s | — | |
| 3 | 500 kbit/s | 0x212, 0x325 | |
| 4 | 500 kbit/s | 0x351, 0x355, 0x356, 0x35A, 0x35E, 0x35F, 0x370–0x381, 0x100, 0x111, 0x200 | The 0x35x set is the common low-voltage inverter CAN set (charge limits 0x351, SOC/SOH 0x355, voltage/current/temperature 0x356, alarms 0x35A, name 0x35E). |
| 5 | 500 kbit/s | 0x305, 0x373, 0x111, 29-bit identifiers | 0x305 is the inverter keep-alive used by several inverter brands. |
| 6 | 500 kbit/s | same as 0 | Protocol 0 at 500 kbit/s |
| 7, 8 | 500 kbit/s | — | |
| 9 | 500 kbit/s | 29-bit identifiers | |
| 10 | 100 kbit/s | — | |
| 11 | 500 kbit/s | 29-bit identifiers | |
| 12–19 | 250 kbit/s | — | Empty slots |

The CAN frames carry values from the telemetry block (0x1200): pack voltage,
current, SOC, SOH, temperatures, alarms, and the charge limits from RCV/RFV and the
current limits. The CAN protocol library version is the high byte of 0x1506
(1 on the test device). ✔

### 3.2 Choosing a CAN protocol

1. Read the enable bitmap at 0x14C4 to see which numbers this board allows.
2. Write the number into the low byte of 0x14B2, together with the current high
   byte (the RS485 protocol). Example: keep RS485 = 1 and select CAN 4 →
   write `01 04` to 0x14B2. ○
3. Set the same bit rate on the inverter.

## 4. Pack address and parallel packs

| Item | Behaviour | |
|---|---|---|
| Pack address | Set with the DIP switches. Shown at 0x1108. Cannot be written over Modbus or Bluetooth. | ✔ |
| Modbus slave address | Equal to the pack address. The BMS answers only its own address; it does not answer broadcast (0) or other addresses. | ✔ |
| Bluetooth name suffix | `-NN` = pack address | ✔ |
| Address 0 | The pack is the parallel master. Its second serial port runs protocol 21 to collect data from the other packs. | ○ |

Several packs on one RS485 bus: give each pack a different address (1–15) and poll
them one after the other from one master.

## 5. Service mode

The BMS has a service mode for factory test. It is entered with the service command
sequence (0x162A / JK02 0xFE). In service mode the RS485 replies always use address 1,
some ports change protocol, and the Bluetooth module can be bridged to a serial
port. Client software must not send these commands.

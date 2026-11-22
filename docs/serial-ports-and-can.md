# Serial ports and CAN

The JK-PB BMS has several communication ports. Each serial port and the CAN port
runs one protocol from a numbered list. The protocol number decides the baud rate
and the frame format on that port.

✔ = checked on a device. ○ = expected behaviour, not yet checked on a device. ? = assumed meaning, not confirmed.

## 1. Ports and protocol selectors

| Port | Protocol number | Selector | Writable |
|---|---|---|---|
| First serial port | 1 on the test device | 0x14B2 high byte | yes ✔ (0, 4, back to 1) |
| CAN | see section 3 | 0x14B2 low byte | yes ○ |
| Second serial port | 1 on the test device | 0x14D4 high byte | no ✔ |
| Third serial port | — | 0x1508 | yes ○ |
| Bluetooth module | JK02 frames, fixed | — | — |

On the test device the RS485 connector that answered Modbus is **not** controlled by
0x14B2: with the 0x14B2 high byte set to 0 and then to 4 (9600 baud), Modbus kept
answering at 115200 on that connector. ✔ Its protocol is probably the second serial
port (0x14D4 = 1) ?. A change of 0x14B2 may also take effect only after a restart ?.
0x14B2 was written over Bluetooth (code 0xC0) and set back to 1.

A protocol number can be selected only if its bit is set in the enable bitmap for
that port. The vendor app builds its protocol drop-down lists from these bitmaps:

| Bitmap | Port | Example |
|---|---|---|
| 0x14B4 | first serial port | `FB E6` = protocols 0, 1, 3–7, 9, 10, 13–15 ✔ |
| 0x14D5 | second serial port | `FB E6` (same list) ✔ |
| 0x14C4 | CAN | `FF 0F` = protocols 0–11 ✔ |
| 0x1509 | third serial port | |

Bit *n* of the bitmap (byte n/8, bit n%8) enables protocol *n*.

Which serial connector on the board belongs to which port can differ between board
revisions. Check it with the vendor manual for your board. ○

**Do not change the protocol of the port you are connected to.** If you change it
from protocol 1 to another number, the Modbus slave stops answering on that port,
and you must change it back from the display or the app.

## 2. Serial protocol list

Each number selects a built-in protocol. Names are the vendor client's own
(English build). All serial protocols use 8N1. Baud is from the firmware ○ except
number 1 ✔. Numbers shown as placeholders are unused empty slots.

| No. | Baud | Protocol |
|---|---|---|
| 0 | 115200 | 4G-GPS Remote module Common protocol V4.2 |
| **1** | **115200** | **JK BMS RS485 Modbus V1.0** (the slave this repo documents — see [Modbus RTU](modbus-rtu.md)) ✔ |
| 2 | 9600 | NIU U SERIES |
| 3 | 9600 | China tower shared battery cabinet V1.1 |
| 4 | 9600 | PACE_RS485_Modbus_V1.3 |
| 5 | 115200 | PYLON_low_voltage_Protocol_RS485_V3.5 |
| 6 | 9600 | Growatt_BMS_RS485_Protocol_1xSxxP_ESS_Rev2.01 |
| 7 | 9600 | Voltronic_Inverter_and_BMS_485_communication_protocol_20200325 |
| 8 | 9600 | China tower shared battery cabinet V2.0 |
| 9 | 9600 | WOW_RS485_Modbus_V1.3 |
| 10 | 115200 | JK BMS LCD Protocol V2.0 |
| 11 | 9600 | UART1 User customization |
| 12 | 9600 | UART2 User customization |
| 13 | 9600 | (9600)JK BMS RS485 Modbus V1.0 |
| 14 | 9600 | (9600)PYLON_low_voltage_Protocol_RS485_V3.5 |
| 15 | 115200 | JK BMS PBxx SERIES LCD Protocol V1.0 |
| 16 | 9600 | JK BMS LIN BUS V1.0 |
| 17 | 9600 | *(empty slot)* |
| 18 | 9600 | *(empty slot)* |
| 19 | 9600 | *(empty slot)* |
| 20 | 9600 | *(empty slot)* |
| 21 | 115200 | Set on the second serial port when the pack address is 0 (parallel master ?). Not in the client dropdown. |

## 3. CAN

### 3.1 CAN protocol list

Each CAN protocol sets its own bit rate. Identifiers are 11-bit unless noted. The bit
rates and identifiers are not yet checked on a device ○, and the notes on inverter
ID sets are assumptions ?.

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
   byte (the first serial port protocol). Example: keep it = 1 and select CAN 4 →
   write `01 04` to 0x14B2. ○
3. Set the same bit rate on the inverter.


### CAN protocol numbers

Names from the vendor client (English build); placeholders are unused slots. [client]

| No. | Protocol |
|---|---|
| 0 | JK BMS CAN Protocol (250K) V2.0 |
| 1 | Deye Low-voltage hybrid inverter CAN communication protocol V1.0 |
| 2 | PYLON-Low-voltage-V1.2 |
| 3 | Growatt BMS CAN-Bus-protocol-low-voltage_Rev_05 |
| 4 | Victron_CANbus_BMS_protocol_20170717 |
| 5 | MEGAREVO_Hybird_BMSCAN_Protocol_V1.0 |
| 6 | JK BMS CAN Protocol (500K) V2.0 |
| 7 | INVT BMS CAN Bus protocol V1.02 |
| 8 | GoodWe LV BMS Protocol (EX/EM/S-BP/BP) |
| 9 | FSS-ConnectingBat-TI-en-10 \| Version 1.0 |
| 10 | MUST PV1800F-CAN communication Protocol1.04.04 |
| 11 | LuxpowerTek Battery CAN protocol V01 |
| 12 | CAN BUS User customization |
| 13 | CAN BUS User customization 2 |
| 14 | *(empty slot)* |
| 15 | *(empty slot)* |
| 16 | *(empty slot)* |
| 17 | *(empty slot)* |
| 18 | *(empty slot)* |
| 19 | *(empty slot)* |
| 20 | *(empty slot)* |

## 4. Pack address and parallel packs

| Item | Behaviour | |
|---|---|---|
| Pack address | Set with the DIP switches. Shown at 0x1108. Cannot be written over Modbus or Bluetooth. | ✔ |
| Modbus slave address | Equal to the pack address. The BMS answers only its own address; it does not answer broadcast (0) or other addresses. | ✔ |
| Bluetooth name suffix | `-NN` = pack address | ✔ |
| Address 0 | The host (master) of a parallel set; addresses 1–15 are slaves. This comes from summaries of the vendor manual and from owner reports, which also say that the host is the BMS connected to the inverter. The second serial port changes to protocol 21, probably to collect the data of the other packs. Not checked on a device (the test pack has address 1). | ○ |

Several packs on one RS485 bus: give each pack a different address (1–15) and poll
them one after the other from one master.

## 5. Service mode

The BMS has a service mode for factory test. It is entered with the service command
sequence (0x162A / JK02 0xFE). In service mode the RS485 replies always use address 1,
some ports change protocol, and the Bluetooth module can be bridged to a serial
port. Client software must not send these commands.

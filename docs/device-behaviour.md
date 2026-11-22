# Device behaviour

How the JK-PB BMS reacts to measurements and to writes. Addresses refer to the
[register map](register-map.md).

✔ = checked on a device. ○ = expected behaviour, not yet checked on a device. ? = assumed meaning, not confirmed.

## 1. Protection

| Protection | Trips when | Action | Releases when |
|---|---|---|---|
| Cell over-voltage | any cell > OVP (0x100C) | charge MOS off | all cells < OVP recovery (0x1010) |
| Cell under-voltage | any cell < UVP (0x1004) | discharge MOS off | all cells > UVP recovery (0x1008) |
| Power-off | any cell < power-off voltage (0x1028) | board turns itself off | — (needs a charger or a button to wake) |
| Charge over-current | current > 0x102C for 0x1030 s | charge MOS off | after 0x1034 s |
| Discharge over-current | current > 0x1038 for 0x103C s | discharge MOS off | after 0x1040 s |
| Short circuit | hardware detection after 0x1080 µs | both MOS off | after 0x1044 s |
| Charge over-temperature | battery temperature > 0x104C | charge MOS off | < 0x1050 |
| Charge under-temperature | battery temperature < 0x105C | charging blocked | > 0x1060 |
| Discharge over-temperature | battery temperature > 0x1054 | discharge MOS off | < 0x1058 |
| Discharge low temperature ? | battery temperature < 0x1122 high byte | discharge blocked | > 0x1122 low byte |
| MOSFET over-temperature | MOSFET temperature > 0x1064 | both MOS off | < 0x1068 |

While a protection waits for its release time, the countdown is shown in
0x12C4–0x12CE. Active protections and faults are flags in 0x12A0. The MOS state is
0x12C0 (high byte charge, low byte discharge). ✔ (registers read; protections not
triggered on purpose)

The user switches 0x1070 (charge) and 0x1074 (discharge) work on top of the
protection: a MOS is on only when its switch is 1 and no protection is active.

## 2. SOC

### 2.1 Normal operation

- The BMS counts charge in and out (coulomb counting) and keeps the remaining
  capacity in 0x12A8 and the full-charge capacity in 0x12AC. SOC (0x12A7) is
  remaining / full. ✔
- When the highest cell stays at or above the SOC 100 % voltage (0x1018) for 4 s, the
  remaining capacity is set to full. Exception: a pack with address 0 and charging
  float mode on sets it to full at the change from Absorption to Float instead. When
  the lowest cell stays at or below the SOC 0 % voltage (0x101C) for 4 s, it is set
  to 0. Counting alone keeps SOC between 1 % and 99 %. ○ Details and pictures:
  [How the BMS charges: stages and SOC](charging-and-soc.md).
- 0x12B0 counts cycles, and 0x12B4 adds up the charged capacity over the lifetime. ✔

### 2.2 SOC re-estimate

Some writes make the BMS discard the coulomb counter and estimate the remaining
capacity from the average cell voltage:

```
full capacity  = nominal capacity (0x107C)
remaining      = nominal capacity × (average cell − UVP) / (OVP − UVP)
                 (0 if average cell ≤ UVP)
```

Checked on a device ✔ (UVP 2800 mV, OVP 3600 mV, nominal capacity 100000 mAh):

| Write | Average cell | Remaining before | Remaining after | Formula |
|---|---|---|---|---|
| 0x107C (write test, +1 then restored) | 3329 mV | 99915 mAh (100 %) | 66125 mAh (66 %) | (3329 − 2800) / 800 × 100000 = 66125 |
| 0x1018 with the same value | 3426 mV | 99997 mAh (100 %) | 99997 mAh (100 %) | no change |
| 0x107C with the **same** value | 3426 mV | 99997 mAh (100 %) | 78250 mAh (78 %) | (3426 − 2800) / 800 × 100000 = 78250 |

The re-estimate is triggered by:

| Trigger | |
|---|---|
| A write of the nominal capacity 0x107C, also with the same value | ✔ |
| A chemistry preset command (0x160A / 0x160C / 0x160E) | ✔ (0x160C) |
| The bulk configuration write of the vendor app | ○ |
| Start-up with an invalid stored remaining capacity | ○ |

After a restart with a valid stored value, SOC and the remaining capacity are kept
(78 % / 78245 mAh before and after a restart with the button). ✔

A write of the SOC 100 % voltage or the SOC 0 % voltage does **not** trigger it ✔.

The estimate is a straight line between UVP and OVP. It is far from the real SOC on
the flat LiFePO4 curve. The value corrects itself at the next full charge. Client
software must write 0x107C only when the user changes the capacity.

The SOC calibration command (0x1630 / JK02 0x6E) is not supported on V19.30 (exception 02
over Modbus, status 2 over Bluetooth). ✔ There is no way to set SOC from outside on this
version: it corrects itself at the next full charge.

## 3. Writes

- A write is applied at once and saved to EEPROM. There is no save command. ✔ (a
  changed value was kept after a restart)
- The same rules apply to writes from Modbus, Bluetooth and the display.
- Some writes have side effects:

| Write | Side effect |
|---|---|
| 0x107C nominal capacity | SOC re-estimate (section 2.2) ✔ |
| 0x1070 / 0x1074 charge / discharge switch | The MOS turns off at once when written 0 ✔. After 1, the charge MOS comes back at once and the discharge MOS within a few seconds ✔. An event is logged; turning discharge on clears the related fault ○. |
| 0x1118 smart-sleep delay | Restarts the sleep countdown 0x1308 (= hours × 3600), also with the same value. ✔ |
| 0x1114 bit 7 | Clears the parallel current-limit fault. ○ |
| 0x1428 Bluetooth name | The module is renamed and restarted; the Bluetooth link drops. ○ |
| 0x1470 setup passcode | Session state is re-initialised. ○ |
| 0x1506 low byte (emergency duration) | Arms the emergency timer with the new duration, also when you only change the duration. ✔ Send 0x1610 afterwards to cancel it. |

- Do not write in a loop. Each write wears the EEPROM.
- A Bluetooth write and a Modbus write at the same moment can lose one EEPROM save.
  Write from one side at a time. ○

## 4. Chemistry presets

The preset commands copy a complete settings block (all fields except the pack
address), save it and reset SOC. ✔ (LiFePO4 preset 0x160C loaded on a device: every
LiFePO4 value in the table below was read back, and SOC was re-estimated with the new
UVP, OVP and capacity. Li-ion and LTO ○.) Over Modbus, the command also sends the new
JK02 settings frame (type 0x01) on RS485 before the FC10 reply. ✔

To undo a preset, write every field back one by one in an order that keeps the
cross-field rules true. The last write of 0x107C re-estimates SOC once more.

| Field | Address | LiFePO4 | Li-ion (NMC) | LTO |
|---|---|---|---|---|
| Smart-sleep voltage | 0x1000 | 3500 | 4150 | 2500 |
| UVP | 0x1004 | 2500 | 2820 | 1800 |
| UVP recovery | 0x1008 | 2900 | 2850 | 1850 |
| OVP | 0x100C | 3650 | 4200 | 2700 |
| OVP recovery | 0x1010 | 3400 | 4170 | 2640 |
| SOC 100 % | 0x1018 | 3445 | 4180 | 2650 |
| SOC 0 % | 0x101C | 2700 | 2840 | 1840 |
| RCV | 0x1020 | 3450 | 4190 | 2680 |
| RFV | 0x1024 | 3350 | 4100 | 2550 |
| Power-off | 0x1028 | 2490 | 2800 | 1700 |
| Balance start | 0x1084 | 3000 | 3000 | 2000 |

Values that are the same in all three presets:

| Field | Value |
|---|---|
| Balance trigger delta | 10 mV |
| Charge / discharge over-current | 200000 mA |
| Delays (charge OCP, charge OCP recovery, discharge OCP, discharge OCP recovery, SCP recovery) | 3 / 60 / 300 / 60 / 5 s |
| Max balance current | 2000 mA |
| Charge over-temperature / recovery | 70.0 / 60.0 °C |
| Discharge over-temperature / recovery | 70.0 / 60.0 °C |
| Charge under-temperature / recovery | −10.0 / 0.0 °C |
| MOSFET over-temperature / recovery | 80.0 / 70.0 °C |
| Cell count | 16 |
| Charge / discharge / balance switches | on / on / on |
| Nominal capacity | 200000 mAh |
| Short-circuit delay | 5 µs |
| Pre-charge time | 5 s |
| Function switches 0x1114 | 0x1010 (display always on, discharge OCP 2) |
| Heating start / stop | −10 / 0 °C |
| Smart-sleep delay | 24 h |
| Discharge low-temperature protection / recovery | −35 / −30 °C |

## 5. Charging voltage, float and inverter limits

- The BMS asks the inverter (over CAN or the inverter serial protocol) for
  RCV × cell count as the charge voltage. ○
- With charging float mode on (0x1114 bit 9), the BMS goes through Bulk, Absorption
  and Float: Absorption starts when the pack is above RCV × cell count and lasts the
  RCV time (0x1504 high byte, × 0.1 h) ✔; then the request changes to RFV × cell count
  if the pack is above SOC 100 % voltage × cell count ✔. Float has no time limit and
  ends when SOC falls by Re-Bulk SOC (0x1510 high byte) ✔. The stage and the time in
  it are in 0x1312 / 0x1310 ✔. Pictures and the exact logic:
  [How the BMS charges: stages and SOC](charging-and-soc.md).
- Charge and discharge current limits sent to the inverter come from the
  over-current settings. ○

## 6. Balancing

- The balancer runs when it is enabled (0x1078), the cells are above the balance
  start voltage (0x1084), and max − min cell > balance trigger delta (0x1014). ○
- The balance current is limited to 0x1048 (300–2000 mA) and shown in 0x12A4. The
  balance state is the high byte of 0x12A6. ✔ (registers)
- Balance-lead resistance is measured per cell (0x124A…). A lead above the limit sets
  its bit in 0x128C and alarm bit 0. ✔ (registers)

## 7. Heating

- With heating enabled (0x1114 bit 0), the heater turns on below the start temperature
  (0x1116 high byte) and off above the stop temperature (0x1116 low byte). ○
- The heater current is shown in 0x12E6 (0 when off). ✔

## 8. Emergency start

- Command 0x1610 turns the discharge MOS on for the emergency duration (0x1506 low
  byte, minutes) even while a protection is active ○. A second command cancels it ✔.
  (With no protection active, 0x12D4 started at 30 × 60 s and went to 0 after the
  second command.)
- The remaining time is shown in 0x12D4. ✔ (register)
- Use it only to start an inverter from a battery that tripped on under-voltage.

## 9. Sleep

- With smart sleep on (0x1114 bit 6), the BMS goes to low-power sleep when the cells
  are at or below the smart-sleep voltage (0x1000) and there is no current for the
  smart-sleep delay (0x1118, hours). ○
- The countdown is shown in 0x1308 (seconds). It counts down 1 per second while smart
  sleep is on, and stays at hours × 3600 while it is off. ✔

## 10. Dry contacts and buzzer

- Dry contact 1, dry contact 2 and the display buzzer each watch one source (0x14E4,
  0x14E6) and switch on when the source crosses the trigger value and off at the
  release value (0x14E8–0x14FC). ✔ (values written and read back)
- The source numbers are listed in the vendor app.
- With "dry-contact alarm intermittent" (0x1114 bit 11), the contact pulses instead of
  staying closed. ○

## 11. Calibration

- **Voltage:** write the real pack voltage (mV) to 0x1600. The BMS computes
  new factor = old factor (0x12DC) × written mV / measured mV (0x1290). If the new
  factor is outside 2.95–3.05, the BMS rejects the write with exception 03 and keeps
  the old factor. The limit is on the absolute factor, not on the change: with a
  factor of 2.995 you can correct only about −1.5 % to +1.8 %. ✔ (V19.30: 3.06, 3.10,
  2.94 and 2.90 rejected, factor unchanged)
- **Current:** at zero current, write 0 to 0x1606 to set the zero offset. Under a known
  current, write the real current (mA) to set the gain. A gain more than 20 % away
  from nominal is rejected. ○
- The full factory reset (0x161A) erases the calibration. Do not use it.

## 12. Clock and logs

- 0x1300 is a real-time clock in seconds since 2020-01-01 00:00. Set it with 0x1612. ✔
  (clock read, decoded and set)
- A restart increases the power-on count (0x1424) by 1. The run time (0x1420) and the
  clock keep counting. Settings are kept. ✔
- The BMS keeps an event log (switch changes, protections). Read it over Bluetooth with
  0xA7 (type 0x06 frames) and 0xA1 (type 0x05), length byte 2, or over Modbus with
  0x1622 and 0x1624. ✔ (frames received; the record fields are not decoded here)
- With timed data logging on (0x1114 bit 8), data is stored every 0x1500 seconds. ○

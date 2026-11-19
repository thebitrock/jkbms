# JK-BMS capability catalogue (from the vendor client model)

Every field the JK-BMS exposes, as declared by the official PC/mobile client's own
data model (decrypted from `en_US.jsonds`, cross-checked with the firmware). This is
the **complete** list of what the device can report and accept; the
[register map](register-map.md) is the address-oriented, hardware-verified view of the
same data and takes precedence where the two differ.

Provenance: names, units, scales, ranges and enum labels here are the vendor client's
own, marked **[client]**. Where the firmware enforces a different (usually tighter)
rule than the client's UI range, the firmware wins — notable cases are called out.

- **Scale**: raw wire value × scale = engineering value in the listed unit.
  A field with scale 0.001 and unit V is sent in mV; 0.1 / °C is sent in 0.1 °C, etc.
- **Type**: wire type (`u8/u16/u32/i8/i16/i32/f32`); `[N]` = array of N elements.
- Bitmask and enum fields list their bit/value meanings underneath. A bitmask entry
  written `label:cleared;set` gives the client's text for the bit = 0 and the bit = 1
  states (a few of the vendor's `set` labels are copy-paste glitches, e.g. `sysAlarm`
  bit 1 is "MOS over temp" regardless of the stray "Over Voltage" label — trust the
  description before the colon).
- Fields and enum labels the vendor model carries only in Chinese are kept verbatim.

## Firmware limits that are tighter than the client UI range

The ranges below are the client's input ranges; the **firmware rejects** values the
client would let you type. The firmware limit is authoritative (verified by decompiling
the write validator; ✔ = also confirmed on a device):

| Field | Client range | Firmware accepts |
|---|---|---|
| Max balance current (`curBalanMax`) | 0.3–15 A | **0.3–2.0 A** (2001 mA → exception 03 ✔) |
| Cell count (`cellCount`) | 2–32 | **2–16** |
| Continued charge current (`timBatCOC`) | 1–600 A | **1–200 A** (1000–200000 mA) |
| Continued discharge current (`timBatDcOC`) | 1–1200 A | **1–200 A** |
| SCP delay (`scpDelay`) | 0–1 000 000 µs | **0–50 µs** |
| Battery capacity (`capBatCell`) | 2–20000 Ah | **2–2000 Ah** |
| Discharge OCP delay (`timBatDcOCPDly`) | 2–600 s | **10–600 s** |

See the [register map](register-map.md) for the per-address firmware rules and the
full validation chain.

## Frame 01 — Settings — read (0x1000-0x1123, JK02 frame 0x01)

Protection thresholds, limits and switches. Stored in EEPROM. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| Vol. Smart Sleep | `volSmartSleep` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell UVP | `volCellUV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell UVPR | `volCellUVPR` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell OVP | `volCellOV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell OVPR | `volCellOVPR` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Balance Trig. Volt. | `volBalanTrig` | u32 | V | 0.003..1 | 0.001 | 0.003 |
| SOC-100% Volt. | `volSOCP100` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| SOC-0% Volt. | `volSOCP0` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Vol. Cell RCV | `volCellRCV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Vol. Cell RFV | `volCellRFV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Power Off Vol. | `volSysPwrOff` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Continued Charge Curr. | `timBatCOC` | u32 | A | 1..600 | 0.001 | 1 |
| Charge OCP Delay | `timBatCOCPDly` | u32 | s | 2..600 |  | 2 |
| Charge OCPR Time | `timBatCOCPRDly` | u32 | s | 2..600 |  | 2 |
| Continued Discharge Curr. | `timBatDcOC` | u32 | A | 1..1200 | 0.001 | 1 |
| Discharge OCP Delay | `timBatDcOCPDly` | u32 | s | 2..600 |  | 2 |
| Discharge OCPR Time | `timBatDcOCPRDly` | u32 | s | 2..600 |  | 2 |
| SCPR Time | `timBatSCPRDly` | u32 | s | 2..600 |  | 2 |
| Max Balance Cur. | `curBalanMax` | u32 | A | 0.3..15 | 0.001 | 2 |
| Charge OTP | `tmpBatCOT` | i32 | °C | 30..80 | 0.1 | 30 |
| Charge OTPR | `tmpBatCOTPR` | i32 | °C | 30..80 | 0.1 | 30 |
| Discharge OTP | `tmpBatDcOT` | i32 | °C | 30..80 | 0.1 | 30 |
| Discharge OTPR | `tmpBatDcOTPR` | i32 | °C | 30..80 | 0.1 | 30 |
| Charge UTP | `tmpBatCUT` | i32 | °C | -45..20 | 0.1 |  |
| Charge UTPR | `tmpBatCUTPR` | i32 | °C | -45..20 | 0.1 |  |
| MOS OTP | `tmpMosOT` | i32 | °C | 50..110 | 0.1 | 50 |
| MOS OTPR | `tmpMosOTPR` | i32 | °C | 50..110 | 0.1 | 50 |
| Cell Count | `cellCount` | u32 |  | 2..32 |  | 2 |
| Charge Enabled | `batChargeEn` | u32 |  | 0..1 |  |  |
| Discharge Enabled | `batDischargeEn` | u32 |  | 0..1 |  |  |
| Balance Enabled | `balanEn` | u32 |  | 0..1 |  |  |
| Battery Capacity | `capBatCell` | u32 | Ah | 2..20000 | 0.001 | 5 |
| SCP Delay | `scpDelay` | u32 | μs | 0..1000000 |  | 10 |
| Start Balance Volt. | `volStartBalan` | u32 | V | 1.2..4.25 | 0.001 | 1.2 |
| Con. Wire Res. | `cellConWireRes` | u32[32] | mΩ | 0..2000 | 0.001 |  |
| Device Addr. | `devAddr` | u32 |  | 0..65535 |  |  |
| Dischrg. Pre. Chrg. T | `dischrgPreChrgT` | u32 | s | 0..300 |  |  |
| Current Range | `currentRange` | u32 | A | 100..20000 | 0.001 | 200 |
| Multiplex Switch Status | `switchStatus` | [16] |  |  |  |  |
| TMP Start Heating | `tmpStartHeating` |  | °C | -40..100 |  |  |
| TMP Stop Heating | `tmpStopHeating` |  | °C | -40..100 |  |  |
| Time Smart Sleep | `timeSmartSleep` | u8 | h | 1..100 |  |  |
| Enable Flags | `enableFlags` | [9] |  |  |  |  |
| Discharge UTP | `tmpBatDCHUT` |  | °C | -40..100 |  |  |
| Discharge UTPR | `tmpBatDCHUTPR` |  | °C | -40..100 |  |  |

**`balanEn` — Balance Enabled** (bit/value → meaning):

- 0 = Off
- 1 = On

**`devAddr` — Device Addr.** (bit/value → meaning):

- 1 = 开启

**`dischrgPreChrgT` — Dischrg. Pre. Chrg. T** (bit/value → meaning):

- 0 = 关闭
- 1 = 开启

**`currentRange` — Current Range** (bit/value → meaning):

- 0 = 关闭
- 1 = 开启

**`switchStatus` — Multiplex Switch Status** (bit/value → meaning):

- 0 = Heating Status:OFF;ON
- 1 = Disable Temp. Sensor:OFF;ON
- 2 = GPS Heartbeat:OFF;ON
- 3 = Multiplexed Port Switching:CAN;RS485
- 4 = Display Always On:OFF;ON
- 5 = Special Charger:OFF;ON
- 6 = Smart Sleep:OFF;ON
- 7 = Disable Par-Limiter:OFF;ON
- 8 = Timed Stored Data:OFF;ON
- 9 = Charging Float Mode:OFF;ON
- 10 = Button Trg. Emerg.:OFF;ON
- 11 = DRY ARM Intermittent:OFF;ON
- 12 = Discharge OCP Ⅱ:OFF;ON
- 13 = Discharge OCP Ⅲ:OFF;ON
- 14 = GPS Locked CHG:OFF;ON
- 15 = GPS Locked DCH:OFF;ON

## Frame 02 — Telemetry — read (0x1200-0x1323, JK02 frame 0x02)

Live measurements, status and alarm words; the BMS streams this frame. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| Cell Voltage | `cellVol` | u16[32] | V |  | 0.001 |  |
| Cell Status | `cellStatus` | [32] |  |  |  |  |
| Ave. Cell Volt. | `cellVolAve` | u16 | V |  | 0.001 |  |
| Cell Volt. Diff. | `maxVoltDelta` | u16 | V |  | 0.001 |  |
| 最高单体 | `celMaxVol` | u8 |  |  |  |  |
| 最低单体 | `celMinVol` | u8 |  |  |  |  |
| 均衡线电阻 | `cellWireRes` | u16[32] | Ω |  | 0.001 |  |
| MOS Temp. | `tempMos` | i16 | °C |  | 0.1 |  |
| 均衡线电阻状态 | `cellWireResStat` | [32] |  |  |  |  |
| Battery Volt. | `batVol` | i32 | V | 0..200 | 0.001 |  |
| Battery Power | `batWatt` | u32 | W |  | 0.001 |  |
| Battery Current | `batCurrent` | i32 | A | -200..200 | 0.001 |  |
| Battery Temp. 1 | `batTemp1` | i16 | °C |  | 0.1 |  |
| Battery Temp. 2 | `batTemp2` | i16 | °C |  | 0.1 |  |
| 系统报警标志 | `sysAlarm` | [32] |  |  |  |  |
| Balance Curr. | `equCurrent` | i16 | A |  | 0.001 |  |
| Balance Status | `equStatus` |  |  |  |  |  |
| Remain Battery | `socRelativeStateOfCharge` | u8 | % | 0..100 |  |  |
| Remain Capacity | `socCapabilityRemain` | u32 | Ah |  | 0.001 |  |
| Battery Capacity | `socFullChargeCapacity` | u32 | Ah |  | 0.001 |  |
| Cycle Count | `socCycleCount` | u32 |  |  |  |  |
| Cycle Capacity | `socCycleCapacity` | u32 | Ah |  | 0.001 |  |
| SOCSOH | `sOCSOH` | u8 | % |  |  |  |
| Pre-Discharge | `preDischarge` |  |  |  |  |  |
| 用户层报警 | `userAlarm` | u16 |  |  |  |  |
| 运行时间 | `runtime` | u32 | s |  |  |  |
| Charge | `chargeStatus` |  |  |  |  |  |
| Discharge | `dischargeStatus` |  |  |  |  |  |
| 用户层报警2 | `userAlarm2` | [16] |  |  |  |  |
| 放电过流保护解除时间 | `timeDcOCPR` | u16 | s |  |  |  |
| 放电短路保护解除时间 | `timeDcSCPR` | u16 | s |  |  |  |
| 充电过流保护解除时间 | `timeCOCPR` | u16 | s |  |  |  |
| 充电短路保护解除时间 | `timeCSCPR` | u16 | s |  |  |  |
| 单体欠压保护解除时间 | `timeUVPR` | u16 | s |  |  |  |
| 单体过压保护解除时间 | `timeOVPR` | u16 | s |  |  |  |
| 传感器存在标志 | `tempSensorAbsent` |  |  |  |  | 255 |
| Heating Status | `heatingStatus` |  |  |  |  | 1 |
| Time Emerg. | `timeEmerg` | [16] |  |  |  |  |
| Discharg. Curr. Correct | `dischrgCurCorrect` | u16 |  |  |  |  |
| Volt. Charg. Curr. | `volChargCur` | u16 | V |  | 0.001 |  |
| Volt. Discharg. Curr. | `volDischargCur` | u16 | V |  | 0.001 |  |
| Bat. Volt. Correct | `batVolCorrect` | f32 |  |  |  |  |
| Charg. PWM | `chargPWMDutyCyle` | u16 | % |  |  |  |
| Discharg. PWM | `dischargPWMDutyCyle` | u16 | % |  |  |  |
| Battery Volt. 2 | `totalBatVol` | u16 | V |  | 0.01 |  |
| Heat Current | `heatCurrent` | i16 | A |  | 0.001 |  |
| Relay | `relay` |  |  |  |  |  |
| ACC Status | `aCCStatus` |  |  |  |  |  |
| Special Charg. Sta | `specialChargerSta` |  |  |  |  |  |
| Startup Flag | `startupFlag` | u8 |  |  |  |  |
| Vol C- | `volC` |  | V |  |  |  |
| MCUID | `mCUID` | u8 |  |  |  |  |
| Charg. Plugged | `chargePlugged` |  |  |  |  |  |
| Sys Run Ticks | `sysRunTicks` | u32 | s |  | 0.1 |  |
| PVD Trig. Time | `pVDTrigTimestamps` | u32 | s |  | 0.1 |  |
| Battery Temp. 3 | `batTemp3` | i16 | °C |  | 0.1 |  |
| Battery Temp. 4 | `batTemp4` | i16 | °C |  | 0.1 |  |
| Battery Temp. 5 | `batTemp5` | i16 | °C |  | 0.1 |  |
| Charg. Curr. Correct | `chrgCurCorrect` | u16 |  |  |  |  |
| RTC Counter | `rtcCounter` | u32 |  |  |  |  |
| Detail Logs Count | `detailLogsCount` | u32 |  |  |  |  |
| Time Enter Sleep | `timeEnterSleep` | u32 | s |  |  |  |
| Par-Limiter | `parLimiter` |  |  |  |  | 1 |
| Cell Type | `batteryType` |  |  |  |  |  |
| Charge Status Time | `chargeStatusTime` | u16 | s |  |  |  |
| Charge Status | `chargeStatus2` |  | s |  |  |  |
| Switch Status | `switchStatus` |  |  |  |  |  |
| Enable Flags | `enableFlags` | [12] |  |  |  |  |

**`cellStatus` — Cell Status** (bit/value → meaning):

- 4 = 1
- 5 = 2
- 6 = 3
- 7 = 4

**`sysAlarm` — 系统报警标志** (bit/value → meaning):

- 0 = Abnormal resistance of the balance wire:Normal;Abnormal
- 1 = Protection - MOS over temp.:Normal;Over Voltage
- 2 = Cell count is not equal to settings:Normal;Over Current
- 3 = Protection - current sensor anomaly:Normal;Over Temp.
- 4 = Battery is fully charged:Not Fully Charged;Fully Charged
- 5 = Protection - Battery over voltage:Normal;Too Large
- 6 = Protection - Charge over current:Normal;Error
- 7 = Protection - Charge short current:Normal;Timeout
- 8 = Protection - Charge over temp.:Normal;Abnormal
- 9 = Protection - Charge under temp.:Normal;Abnormal
- 10 = Protection - CPUAUX anomaly:Normal;Abnormal
- 11 = Protection - Cell under voltage:Normal;Abnormal
- 12 = Protection - Battery under voltage:Normal;Abnormal
- 13 = Protection - Discharge over current:Normal;Abnormal
- 14 = Protection - Discharge short circuit:Normal;Abnormal
- 15 = Protection - Discharge over temp.:Normal;Abnormal
- 16 = Charging MOS not normal:Normal;Abnormal
- 17 = Discharging MOS not normal:Normal;Abnormal
- 18 = GPS disconnected:Normal;Abnormal
- 19 = Modify PWD. in time:Normal;Abnormal
- 20 = Discharge on failed:Normal;Abnormal
- 21 = Battery over temp. alarm:Normal;Abnormal
- 22 = Temp sensor abnormal:Normal;Abnormal
- 23 = PL module abnormal:Normal;Abnormal
- 24 = SCP release failed:Normal;Abnormal
- 25 = DCHOCP2:Normal;Abnormal
- 26 = DCHOCP3:Normal;Abnormal
- 27 = Alarm Dch UTP:Normal;Abnormal
- 28 = GPS Remote Lock:Normal;Abnormal

**`equStatus` — Balance Status** (bit/value → meaning):

- 0 = OFF
- 1 = ON
- 2 = ON
- 3 = ON

**`preDischarge` — Pre-Discharge** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`chargeStatus` — Charge** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`dischargeStatus` — Discharge** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`userAlarm2` — 用户层报警2** (bit/value → meaning):

- 0 = Temperature is Too High:Normal;Not Normal
- 1 = Temperature is Too Low:Normal;Not Normal
- 2 = The Temp. Diff. is Too Large:Normal;Not Normal
- 3 = Cell Voltage is Too High:Normal;Not Normal
- 4 = Cell Voltage is Too Low:Normal;Not Normal
- 5 = Discharge Overcurrent:Normal;Not Normal
- 6 = Charge Overcurrent:Normal;Not Normal
- 7 = Temperature Sensor Failure:Normal;Not Normal
- 8 = Charger Communication Failure:Normal;Not Normal
- 10 = Heating Relay Failure:Normal;Not Normal
- 11 = Charger Failure:Normal;Not Normal
- 12 = Low Temp. Heating Failure:Normal;Not Normal
- 13 = Heater Failure:Normal;Not Normal

**`tempSensorAbsent` — 传感器存在标志** (bit/value → meaning):

- 0 = MOS温度
- 1 = 电池温度1
- 2 = 电池温度2
- 3 = 电池温度3
- 4 = 电池温度4
- 5 = 电池温度5

**`heatingStatus` — Heating Status** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`timeEmerg` — Time Emerg.** (bit/value → meaning):

- 0 = OFF

**`relay` — Relay** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`aCCStatus` — ACC Status** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`specialChargerSta` — Special Charg. Sta** (bit/value → meaning):

- 0 = Not Plugged
- 1 = Plugged

**`chargePlugged` — Charg. Plugged** (bit/value → meaning):

- 0 = Not Plugged
- 1 = Plugged

**`parLimiter` — Par-Limiter** (bit/value → meaning):

- 0 = OFF
- 1 = ON

**`batteryType` — Cell Type** (bit/value → meaning):

- 0 = LFP
- 1 = Lion
- 2 = LTO

**`chargeStatus2` — Charge Status** (bit/value → meaning):

- 0 = Bulk
- 1 = Abs
- 2 = Float

**`switchStatus` — Switch Status** (bit/value → meaning):

- 0 = LCD Buzzer Alarm:OFF;ON
- 1 = DRY1 Alarm:OFF;ON
- 2 = DRY2 Alarm:OFF;ON

## Frame 03 — Device info — read (0x1400-0x1523, JK02 frame 0x03)

Identity, counters, names, port protocols, dry-contact config. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| Vendor ID | `manuDeviceID` | i8[15] |  |  |  |  |
| Max Cells | `maxCells` | u8 |  |  |  |  |
| Hardware Ver | `hardwareVersion` | i8[8] |  |  |  |  |
| Software Ver | `softwareVersion` | i8[8] |  |  |  |  |
| Total Time | `oddRunTime` | u32 | s |  |  |  |
| Power-on Times | `pwrOnTimes` | u32 | count |  |  |  |
| BLE Name | `bluetoothName` | i8[16] |  |  |  |  |
| BLE PWD | `bluetoothPwd` | i8[16] |  |  |  |  |
| First On Date | `manufactureDate` | i8[8] |  |  |  |  |
| Serial Number | `deviceSN` | i8[16] |  |  |  |  |
| User Private Data | `userData` | i8[16] |  |  |  |  |
| Setting Password | `settingPassword` | i8[16] |  |  |  |  |
| User Data 2 | `userData2` | i8[16] |  |  |  |  |
| Cmd Support Flags | `cmdSupportFlags` | [32] |  |  |  |  |
| Agency ID | `agencyId` | u16 |  |  |  |  |
| UART1 Protocol No. | `uart1ProtoNo` | u8 |  |  |  |  |
| CAN Protocol No. | `canProtoNo` | u8 |  |  |  |  |
| UART1 Protocol Enabled | `uart1ProtoEnabled` | [4] |  |  |  |  |
| Hardware Option | `hardwareOption` | i8[12] |  |  |  |  |
| CAN Protocol Enabled | `canProtoEnabled` | [4] |  |  |  |  |
| UART2 Protocol No. | `uart2ProtoNo` | u8 |  |  |  |  |
| UART2 Protocol Enabled | `uart2ProtoEnabled` | [4] |  |  |  |  |
| LCD Buzzer Trigger | `lcdBuzzerTrigger` | u8 |  |  |  |  |
| DRY 1 Trigger | `dry1Trigger` | u8 |  |  |  |  |
| DRY 2 Trigger | `dry2Trigger` | u8 |  |  |  |  |
| UART MPT L Ver | `uartMPTLVer` | u8 |  |  |  |  |
| LCD Buzzer Trigger Val | `lcdBuzzerTriggerVal` | i32 |  |  |  |  |
| LCD Buzzer Release Val | `lcdBuzzerReleaseVal` | i32 |  |  |  |  |
| DRY 1 Trigger Val | `dry1TriggerVal` | i32 |  |  |  |  |
| DRY 1 Release Val | `dry1ReleaseVal` | i32 |  |  |  |  |
| DRY 2 Trigger Val | `dry2TriggerVal` | i32 |  |  |  |  |
| DRY 2 Release Val | `dry2ReleaseVal` | i32 |  |  |  |  |
| Data Stored Period | `dataStoredPeriod` | u32 | s | 5..864000 |  | 5 |
| RCV Time | `rcvTime` | u8 | h | 0..25 | 0.1 |  |
| RFV Time | `rfvTime` | u8 | h | 0..25 | 0.1 |  |
| CAN MPT L Ver | `canMPTLVer` | u8 |  |  |  |  |
| Emerg. Time | `emergencyTime` | u8 | min |  |  |  |
| UART3 Protocol No. | `uart3ProtoNo` | u8 |  |  |  |  |
| UART3 Protocol Enabled | `uart3ProtoEnabled` | [7] |  |  |  |  |
| Re-Bulk SOC | `reBulkSOC` | u8 | % | 0..50 |  |  |
| UART4 Protocol No. | `uart4ProtoNo` | u8 |  |  |  |  |
| UART4 Protocol Enabled | `uart4ProtoEnabled` | [4] |  |  |  |  |
| Enable Flags | `enableFlags` | [11] |  |  |  |  |
| Protocol Version | `protocolVer` | u8 |  |  |  |  |

## Frame 04 — Settings — write (downlink)

The same settings block written back to the BMS. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| Vol. Smart Sleep | `volSmartSleep` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell UVP | `volCellUV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell UVPR | `volCellUVPR` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell OVP | `volCellOV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Cell OVPR | `volCellOVPR` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Balance Trig. Volt. | `volBalanTrig` | u32 | V | 0.003..1 | 0.001 | 0.003 |
| SOC-100% Volt. | `volSOCP100` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| SOC-0% Volt. | `volSOCP0` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Vol. Cell RCV | `volCellRCV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Vol. Cell RFV | `volCellRFV` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Power Off Vol. | `volSysPwrOff` | u32 | V | 1.2..4.4 | 0.001 | 1.2 |
| Continued Charge Curr. | `timBatCOC` | u32 | A | 1..600 | 0.001 | 1 |
| Charge OCP Delay | `timBatCOCPDly` | u32 | s | 2..600 |  | 2 |
| Charge OCPR Time | `timBatCOCPRDly` | u32 | s | 2..600 |  | 2 |
| Continued Discharge Curr. | `timBatDcOC` | u32 | A | 1..1200 | 0.001 | 1 |
| Discharge OCP Delay | `timBatDcOCPDly` | u32 | s | 2..600 |  | 2 |
| Discharge OCPR Time | `timBatDcOCPRDly` | u32 | s | 2..600 |  | 2 |
| SCPR Time | `timBatSCPRDly` | u32 | s | 2..600 |  | 2 |
| Max Balance Cur. | `curBalanMax` | u32 | A | 0.3..15 | 0.001 | 2 |
| Charge OTP | `tmpBatCOT` | i32 | °C | 30..80 | 0.1 | 30 |
| Charge OTPR | `tmpBatCOTPR` | i32 | °C | 30..80 | 0.1 | 30 |
| Discharge OTP | `tmpBatDcOT` | i32 | °C | 30..80 | 0.1 | 30 |
| Discharge OTPR | `tmpBatDcOTPR` | i32 | °C | 30..80 | 0.1 | 30 |
| Charge UTP | `tmpBatCUT` | i32 | °C | -45..20 | 0.1 |  |
| Charge UTPR | `tmpBatCUTPR` | i32 | °C | -45..20 | 0.1 |  |
| MOS OTP | `tmpMosOT` | i32 | °C | 50..110 | 0.1 | 50 |
| MOS OTPR | `tmpMosOTPR` | i32 | °C | 50..110 | 0.1 | 50 |
| Cell Count | `cellCount` | u32 |  | 2..32 |  | 2 |
| Charge Enabled | `batChargeEn` | u32 |  | 0..1 |  |  |
| Discharge Enabled | `batDischargeEn` | u32 |  | 0..1 |  |  |
| Balance Enabled | `balanEn` | u32 |  | 0..1 |  |  |
| Battery Capacity | `capBatCell` | u32 | Ah | 2..20000 | 0.001 | 5 |
| SCP Delay | `scpDelay` | u32 | μs | 0..1000000 |  | 10 |
| Start Balance Volt. | `volStartBalan` | u32 | V | 1.2..4.25 | 0.001 | 1.2 |
| Con. Wire Res. | `cellConWireRes` | u32[32] | mΩ | 0..2000 | 0.001 |  |
| Device Addr. | `devAddr` | u32 |  | 0..65535 |  |  |
| Dischrg. Pre. Chrg. T | `dischrgPreChrgT` | u32 | s | 0..300 |  |  |
| Current Range | `currentRange` | u32 | A | 100..20000 | 0.001 | 1000 |
| Multiplex Switch Status | `switchStatus` | [16] |  |  |  |  |
| TMP Start Heating | `tmpStartHeating` |  | °C | -40..100 |  |  |
| TMP Stop Heating | `tmpStopHeating` |  | °C | -40..100 |  |  |
| Time Smart Sleep | `timeSmartSleep` | u8 | h | 1..100 |  |  |
| Discharge UTP | `tmpBatDCHUT` |  | °C | -40..100 |  |  |
| Discharge UTPR | `tmpBatDCHUTPR` |  | °C | -40..100 |  |  |

**`balanEn` — Balance Enabled** (bit/value → meaning):

- 0 = Off
- 1 = On

**`devAddr` — Device Addr.** (bit/value → meaning):

- 1 = 开启

**`dischrgPreChrgT` — Dischrg. Pre. Chrg. T** (bit/value → meaning):

- 0 = 关闭
- 1 = 开启

**`currentRange` — Current Range** (bit/value → meaning):

- 0 = 关闭
- 1 = 开启

**`switchStatus` — Multiplex Switch Status** (bit/value → meaning):

- 0 = Heating Switch:OFF;ON
- 1 = Disable Temp. Sensor:OFF;ON
- 2 = GPS Heartbeat:OFF;ON
- 3 = Multiplexed Port Function:CAN;RS485
- 4 = Display Always On:OFF;ON
- 5 = Special Charger:OFF;ON
- 6 = Smart Sleep:OFF;ON
- 7 = Disable Par-Limiter:OFF;ON
- 8 = Timed Stored Data:OFF;ON
- 9 = Charging Float Mode:OFF;ON
- 10 = Button Trg. Emerg.:OFF;ON
- 11 = DRY ARM Intermittent:OFF;ON
- 12 = Discharge OCP Ⅱ:OFF;ON
- 13 = Discharge OCP Ⅲ:OFF;ON
- 14 = GPS Locked CHG:OFF;ON
- 15 = GPS Locked DCH:OFF;ON

## Frame 05 — System log (uplink)

The on-device alarm log. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| 日志条数 | `logCount` | u32 |  |  |  |  |
| 校验码 | `check` | u8 |  |  |  |  |
| 报警日志 | `alarmLog` | [250] |  |  |  |  |
| 备用 | `_reserved1` | [38] |  |  |  |  |

## Frame 06 — Fault / event records (uplink)

Historical event records; each record is a snapshot at the moment of the event. [client]

| Field | Key | Type | Unit | Range | Scale | Default |
|---|---|---|---|---|---|---|
| 起始编号 | `beginIndex` | u16 |  |  |  |  |
| 信息条数 | `count` | u8 |  |  |  |  |
| 记录1 | `record1` |  |  |  |  |  |
| RTC时间戳 | `rtcCount` | u32 | s |  |  |  |
| 日志代码 | `logCode` | u8 |  |  |  |  |
| 开关状态 | `switchSta` |  |  |  |  |  |
| 最高单体编号 | `maxVolCellNo` | u8 |  |  |  |  |
| 最低单体编号 | `minVolCellNo` | u8 |  |  |  |  |
| 最高单体电压 | `volCellMax` | u16 | V |  | 0.001 |  |
| 最低单体电压 | `volCellMin` | u16 | V |  | 0.001 |  |
| 电池电压 | `volBat` | u16 | V |  | 0.01 |  |
| 电池电流 | `curBat` | u16 | A |  | 0.1 |  |
| 剩余容量 | `socCapRemain` | u16 | Ah |  | 0.1 |  |
| 实际容量 | `socFullChargeCap` | u16 | Ah |  | 0.1 |  |
| 最高温度 | `maxTemp` |  | °C |  |  |  |
| 最低温度 | `minTemp` |  | °C |  |  |  |
| 功率温度 | `tempMos` |  | °C |  |  |  |
| 加热电流 | `hearCurrent` |  | A |  | 0.1 |  |
| 记录2~12 | `record2_12` | [264] |  |  |  |  |
| 帧校验 | `check` |  |  |  |  |  |

**`switchSta` — 开关状态** (bit/value → meaning):

- 0 = 充电管状态:关闭;打开
- 1 = 放电管状态:关闭;打开
- 2 = 均衡状态:关闭;打开
- 3 = 加热状态:关闭;打开


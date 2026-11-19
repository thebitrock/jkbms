# Firmware files and update (IAP)

How the JK-PB firmware (`.jkbms`) and configuration (`.dsd` / `.dsdx`) files are
encrypted, how the vendor updater turns a `.jkbms` into the raw image, what the image
must contain, and how the device is put into its bootloader to receive it.

This page describes the **file format and the PC-side update flow**. It was worked out
by decompiling the vendor updater, not by flashing a device: the bytes the updater sends
are known completely (section 5), but the bootloader's side (timing, first prompt) is
not. Nothing here was verified by performing an update. **Do not flash with anything but the vendor tool unless you fully understand
the bootloader protocol; a bad or interrupted flash bricks a BMS that is managing a
live battery.**

## 1. File container

Both file kinds use the same cipher:

| Item | Value |
|---|---|
| Cipher | AES-256-CBC |
| Key | 32 ASCII characters, used as the 32-byte key (see below) |
| IV | 16 zero bytes |
| Padding | PKCS#7 |
| Plaintext layout | `[uint32 LE: uncompressed size][zlib stream]` |

After decryption, the first 4 bytes are the uncompressed size (little-endian), and the
rest is a zlib stream. Inflate it to get the payload (the raw Cortex-M flash image for
`.jkbms`, or the settings blob for `.dsd`).

Keys (ASCII, used verbatim as the AES-256 key):

| File kind | Key |
|---|---|
| Firmware `.jkbms` | `A39FF3F613F94FDD957EC22EF642ADA9` |
| Config `.dsd` / `.dsdx` | `2B10F23AC94C4910AE8BCCE19E4D485B` |

- A `.jkbms` file is ciphertext from byte 0; there is no header.
- A `.dsd` / `.dsdx` file has a 14-byte header first: 8-byte magic (`dsd` / `dsdx`),
  1 byte version (0x11 / 0x21), 1 byte encrypt flag (1 = encrypted), 4 bytes LE length.
  The ciphertext follows the header.

### Decrypt (Node.js, no dependencies)

```js
import { readFileSync, writeFileSync } from "node:fs";
import { createDecipheriv } from "node:crypto";
import { inflateSync } from "node:zlib";

const KEYS = {
  firmware: "A39FF3F613F94FDD957EC22EF642ADA9",
  config:   "2B10F23AC94C4910AE8BCCE19E4D485B",
};
// node decrypt.mjs in.jkbms out.bin firmware
const [, , inp, out, kind = "firmware"] = process.argv;
const d = createDecipheriv("aes-256-cbc", Buffer.from(KEYS[kind], "ascii"), Buffer.alloc(16));
let ct = readFileSync(inp);
// for .dsd/.dsdx, strip the 14-byte header first: ct = ct.subarray(14);
const pt = Buffer.concat([d.update(ct), d.final()]);
const size = pt.readUInt32LE(0);
const bin = inflateSync(pt.subarray(4));
if (bin.length !== size) throw new Error(`size mismatch: ${size} vs ${bin.length}`);
writeFileSync(out, bin);
```

The encryption only protects the distributed file. The device does **not** decrypt:
the updater decrypts on the PC and sends the plaintext image to the bootloader.

## 2. Raw image layout (`.jkbms` payload)

The inflated image is an ARM Cortex-M flash image for the PB application. The vendor
updater reads these parts before it will upload:

| Region | Content |
|---|---|
| ASCII strings around offsets 0x200, 0x210, 0x220, 0x230, 0x250 | model id, software version (`Vx`), device-identify string, compatible id |
| ints around offset 0x240 | version / type fields |
| body | the application code+data (vector table base 0x08002000, see note) |
| **last 12 bytes (trailer)** | `[int64 LE: build timestamp, ms since epoch][int32 LE: validity, hours]` |

- The updater requires the image to be at least 0x260 bytes.
- **Validity field.** `validity_h` is **0 on the official release files** (checked on the
  073 images for V19.34 and V19.36), meaning **no expiry** — which matches the vendor's
  download page. The updater only applies a time window (reject if the PC clock is before
  `build_ts − 2 h` or after `build_ts + validity_h`) when `validity_h` is non-zero, which
  is not the case for the public releases. (`build_ts` still decodes to the real build time,
  e.g. V19.36 = 2026-09-07; it is stored in China local time, +8 h, like the RTC.)
- The 12-byte trailer is stripped; the body (image minus 12 bytes) is what is uploaded.

> Note on the base address: the application image loads at 0x08002000. The first
> 0x2000 of flash is the bootloader, which is **not** part of the application image.

## 3. Compatibility checks (against the connected BMS)

Before the upload is armed, the updater compares the file header with the live device
(read over the normal protocol) and refuses on any mismatch:

| Check | Message on failure |
|---|---|
| A type field in the header must equal 2 | "Device Identify is not same to connected device!" |
| Major version of the file must equal the device's major version | "Major version is not same to connected device!" |
| Minor version of the file must be **greater** than the device's minor | "Minor version must be larger than connected device!" |
| Device-identify string of the file must equal the device's | "Device Identify is not same to connected device!" |

So the vendor tool only allows an **upgrade of the same model to a higher minor
version**; it will not downgrade and will not cross models.

## 4. Entering the bootloader

The application is told to reboot into its bootloader with a command (see the
[register map](register-map.md) and [Bluetooth](bluetooth-jk02.md)):

- Modbus / command block: **0x1626**
- JK02 (BLE) command code: **0xFF**

At power-up the bootloader prints a plain-text banner on the RS485 line (observed on a
device), for example:

```
-> JKBMS STM32F103x  bootloader (Version 2.0.5)
-> Build Feb 27 2026 ...
-> IAP Timeout
-> Jump to app...
```

If no download arrives within its timeout, it prints `IAP Timeout` and jumps to the
application. So the bootloader waits for the image on RS485 after the reset.

## 5. Transfer to the bootloader (RS485)

The whole update runs on the RS485 port at the normal line settings (115200 8N1; the
updater does not reconfigure the port). It has two stages: one Modbus RTU command, then
an **XMODEM (128-byte blocks, 8-bit checksum)** download of the image.

This was recovered by decompiling the sender — the vendor updater
`jk-bms-monitor.exe` ("Start/Force Updating"). The sender fully defines the bytes on
the wire, so the framing below is complete from the PC side. It was **not** verified by
performing an update (○); the bootloader (first 0x2000 of flash) is not in the
distributed image, so its timing and its first prompt byte are not known.

### 5.1 Stage 1 — enter-bootloader command (Modbus)

A Modbus RTU **FC 0x10 write of one register at 0x1626, value 0x0000**:

```
[slave] 10 16 26 00 01 02 00 00 [CRC-16/MODBUS lo][hi]
pack address 1:   01 10 16 26 00 01 02 00 00 D6 97
```

The updater does not wait for or parse a normal Modbus reply to this frame. While the
0x1626 request is pending, **every byte received on the port goes to the XMODEM handler
of stage 2** instead of the Modbus parser, with no timeout. The vendor procedure enters
the bootloader with the power + reset buttons (section 4), and the bootloader then
starts the download by sending a control byte.

### 5.2 Stage 2 — image download (XMODEM, checksum variant)

The data sent is the decrypted image **without its 12-byte trailer** (section 2), cut
into `ceil(size / 128)` blocks. Each block:

```
01            SOH
NN            block number = index + 1 (8 bit, wraps after 0xFF)
~NN           NN XOR 0xFF
128 bytes     image[index·128 … index·128+127]; the last block is padded with 0xFF
SS            8-bit sum of the 128 data bytes
```

On the **last** block the updater appends `04 04 04` (EOT × 3) in the same write. It
waits 1 ms before each write.

The updater reacts to the newest control byte it receives (it scans each received
chunk from the end; other bytes, such as the bootloader's text banner, are ignored):

| Byte received | Updater action |
|---|---|
| `06` ACK | All blocks sent → **update finished, success**. Otherwise → send the next block. |
| `15` NAK | Step back one block and send it again (at the start: send block 1). |
| `18` CAN | Abort, update failed. |
| `00` | Ignored (stop scanning this chunk). |

`C` (0x43, the CRC-mode request of XMODEM-CRC) is **not** handled: the download is the
original checksum XMODEM. So the bootloader starts the transfer with NAK (or ACK), and
acknowledges each block.

Example for the 073 V19.36 image: 102 807 bytes → 102 795 bytes sent → 804 blocks; block 1
begins `01 01 FE 78 6F 00 20 …` (the application's initial stack pointer 0x20006F78),
and the last block carries 11 data bytes, 117 × `FF`, its checksum, then `04 04 04`.

### Checksums used across the product

| Link | Checksum |
|---|---|
| Modbus RTU (RS485) | CRC-16/MODBUS: poly 0xA001 (reflected), init 0xFFFF, low byte first |
| JK02 (BLE / RS485 frames) | 8-bit sum of the frame bytes |
| Firmware update, stage 1 (0x1626 command) | CRC-16/MODBUS |
| Firmware update, stage 2 (image blocks) | **8-bit sum of the 128 data bytes** (classic XMODEM checksum) |
| `.jkbms` / `.dsd` file body | none in the body (AES-256-CBC + zlib; a size header and the zlib checksum catch corruption) |

> History: earlier versions of this page first guessed CRC-16/XMODEM (from an unused
> `crc16_xmodem` export in `protocore.dll`), then said the image block framing was
> unknown. Both were wrong: the image goes as checksum XMODEM (no CRC-16), handled in a
> separate receive path of the updater that runs only while the 0x1626 command is pending.

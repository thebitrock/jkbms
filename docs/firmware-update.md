# Firmware files and update (IAP)

How the JK-PB firmware (`.jkbms`) and configuration (`.dsd` / `.dsdx`) files are
encrypted, how the vendor updater turns a `.jkbms` into the raw image, what the image
must contain, and how the device is put into its bootloader to receive it.

This page describes the **file format and the PC-side update flow**. It was worked out
from the vendor updater, not by flashing a device, so the on-wire bootloader transfer
is only partly known (see section 5). Nothing here was verified by performing an
update. **Do not flash with anything but the vendor tool unless you fully understand
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
- **Firmware files expire.** The updater rejects the file ("Binary file is invalid!")
  if the PC clock is earlier than `build_ts − 2 h` or later than
  `build_ts + validity_h`. Set the PC clock correctly, or a valid file is refused.
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

## 5. Transfer to the bootloader (partly known)

- The decrypted, trailer-stripped image is streamed to the bootloader over the serial
  link after the reset.
- **The transfer uses a CRC-16/XMODEM checksum** (polynomial 0x1021, init 0x0000, no
  input/output reflection, no final XOR). This is a different checksum from the two
  runtime protocols — Modbus uses CRC-16/MODBUS and JK02 uses an 8-bit byte sum — which
  points to a block-oriented (XMODEM/YMODEM-style) download.
- **Not published / not fully known:** the exact block size, the block header and
  opcodes, and the acknowledge/retry handshake. These live in the bootloader (the first
  0x2000 of flash), which is not part of the distributed application image. Recovering
  them needs a bootloader dump from a device (for example over SWD).

### Checksums used across the product

| Link | Checksum |
|---|---|
| Modbus RTU (RS485) | CRC-16/MODBUS: poly 0xA001 (reflected), init 0xFFFF, low byte first |
| JK02 (BLE / RS485 frames) | 8-bit sum of the frame bytes |
| Firmware IAP (bootloader) | CRC-16/XMODEM: poly 0x1021, init 0x0000 |
| `.jkbms` / `.dsd` file body | none in the body (AES-256-CBC + zlib; a size header and the zlib checksum catch corruption) |

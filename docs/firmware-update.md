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

## 5. Transfer to the bootloader (RS485 Modbus RTU)

The transfer uses the **normal Modbus RTU transport** — there is no separate
XMODEM/YMODEM download. This was traced through the updater's send path
(`jk-bms-monitor.exe`, "Start/Force Updating"); it is **not** verified by performing an
update, and one step is still open (see below).

**The send path.** Pressing Start/Force Updating hands the decrypted, trailer-stripped
image to one send call, which runs on a background worker with a byte progress bar
(bytes sent / total) and a completion callback `(success, code, sub-code)`. The
frame serializer builds a standard Modbus RTU frame and writes it to the serial channel:

```
[slave addr] [func 0x03 or 0x10] [reg addr hi][lo] [qty hi][lo] ( [byte count][data…] ) [CRC-16/MODBUS lo][hi]
```

- Addresses and quantities are big-endian; function codes are only 0x03 / 0x10; the
  trailing CRC is **CRC-16/MODBUS** (the same checksum as every other RS485 frame).
  There is no byte-stuffing.
- **Entering the bootloader is command 0x1626** (block base 0x1000 + 0x626), sent as the
  reboot step (matches the Modbus 0x1626 / JK02 0xFF in section 4).

**Acknowledgement model.** The transfer waits for Modbus-style replies and maps the
result the same way as a normal register write: exception **01** = function code /
register not supported, **02** = register inaccessible, **03** = data out of
limit, **04** = CRC error, plus worker-level *timeout*, *canceled* and
*not-supported-command*. So the bootloader answers like a Modbus slave.

**Still open:** the per-block framing of the image body **after** the 0x1626 reset — the
block size and how the image bytes are sequenced to the bootloader. In the updater's
recovered command path the image is attached to the 0x1626 request but the serializer
skips that inline payload for the 0x1626 address, so the image-streaming step is not
visible in the PC code we traced; it is consistent with the bootloader (device side, not
in the distributed image) driving the block transfer after the reset. A capture of the
vendor updater on the RS485 line during a real update, or a bootloader dump over SWD,
would show the exact block framing.

### Checksums used across the product

| Link | Checksum |
|---|---|
| Modbus RTU (RS485) | CRC-16/MODBUS: poly 0xA001 (reflected), init 0xFFFF, low byte first |
| JK02 (BLE / RS485 frames) | 8-bit sum of the frame bytes |
| **Firmware IAP transfer** | **CRC-16/MODBUS** — the transfer is plain Modbus RTU (see above), not a separate scheme |
| `.jkbms` / `.dsd` file body | none in the body (AES-256-CBC + zlib; a size header and the zlib checksum catch corruption) |

> History: an early draft of this page guessed CRC-16/XMODEM for the download, on the
> strength of a `crc16_xmodem` export in `protocore.dll`. That was wrong — nothing in
> the build calls it. Tracing the updater's actual send path showed the transfer is
> ordinary Modbus RTU with CRC-16/MODBUS, as above.

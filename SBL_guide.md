````md
# Intel Slim Bootloader IFWI / BIOS / Capsule Update Guide

This guide explains the complete firmware flow for Intel platforms using Slim Bootloader (SBL) and Intel FSP.

---

# 1. Big Picture

On Intel platforms, the firmware image stored in SPI flash is usually called:

- **IFWI** → *Intel Firmware Image*
- It contains:
  - Bootloader / BIOS
  - Intel ME/TXE firmware
  - Descriptor region
  - Microcode
  - NVRAM
  - ACM
  - Other platform firmware components

When using Slim Bootloader (SBL), you replace the normal BIOS boot firmware with SBL.

So the flow becomes:

```text
SPI Flash
│
├── Flash Descriptor
├── Intel ME/TXE
├── GbE region (optional)
└── BIOS Region
      └── Slim Bootloader (SBL)
````

---

# 2. Important Terms

## BIOS

Historically:

* BIOS = firmware that initializes hardware and boots OS.

Modern Intel systems:

* "BIOS image" usually means:

  * the firmware blob inside the BIOS region of IFWI.

---

## IFWI (Intel Firmware Image)

Full SPI flash image.

Contains multiple regions:

```text
+----------------------------------+
| Flash Descriptor                 |
+----------------------------------+
| Intel ME / TXE firmware          |
+----------------------------------+
| GbE firmware (optional)          |
+----------------------------------+
| BIOS Region                      |
|   └── Slim Bootloader / UEFI     |
+----------------------------------+
```

Usually:

* 16 MB
* 32 MB
* 64 MB SPI flash dump

---

## Slim Bootloader (SBL)

A lightweight Intel bootloader.

Stages:

```text
Reset Vector
   ↓
Stage1A
   ↓
Stage1B
   ↓
Stage2
   ↓
Payload (UEFI/Linux/etc)
```

SBL replaces traditional UEFI BIOS firmware.

---

## FSP (Firmware Support Package)

Binary package from Intel.

Provides:

* memory init
* silicon init
* temp RAM init

Components:

```text
FSP-T → TempRamInit()
FSP-M → MemoryInit()
FSP-S → SiliconInit()
```

---

# 3. Firmware Layout

## Factory IFWI

```text
SPI Flash
│
├── Descriptor
├── ME
└── BIOS Region
     └── Vendor BIOS
```

---

## SBL IFWI

After stitching:

```text
SPI Flash
│
├── Descriptor
├── ME
└── BIOS Region
     └── Slim Bootloader
```

---

# 4. Build Slim Bootloader

Go to source tree:

```bash
cd slimbootloader
```

Build for ADL-N board:

```bash
python BuildLoader.py build adln50
```

Output:

```text
Outputs/adln50/SlimBootloader.bin
```

This is ONLY the SBL BIOS payload.

NOT full IFWI.

---

# 5. What is Stitching?

Stitching means:

> Insert SBL into the BIOS region of the factory IFWI.

---

## Before Stitching

```text
Factory IFWI
└── BIOS Region
     └── Vendor BIOS
```

---

## After Stitching

```text
Modified IFWI
└── BIOS Region
     └── Slim Bootloader
```

---

# 6. Stitch SBL Into Factory IFWI

Command:

```bash
python Platform/AlderlakeBoardPkg/Script/StitchLoader.py \
  -i factory_ifwi.bin \
  -s Outputs/adln50/SlimBootloader.bin \
  -o sbl_ifwi.bin \
  -p 0xAA000106
```

---

# 7. Understanding Parameters

## `-i`

Input factory IFWI:

```text
factory_ifwi.bin
```

Usually dumped from board vendor BIOS.

---

## `-s`

SBL binary:

```text
Outputs/adln50/SlimBootloader.bin
```

Generated from build.

---

## `-o`

Final stitched IFWI:

```text
sbl_ifwi.bin
```

---

## `-p`

Platform data.

Example:

```text
0xAA000106
```

Encodes:

* board ID
* UART debug port
* platform-specific config

---

# 8. Important Boot Guard Note

`StitchLoader.py` only works if:

* Boot Guard disabled
* chipset not fused for Boot Guard

If Boot Guard enabled:

* use:

  * `StitchIfwi.py`

Otherwise:

* system may refuse boot.

---

# 9. Flashing the IFWI

You now have:

```text
sbl_ifwi.bin
```

This is a FULL SPI image.

Flash using:

* Dediprog
* CH341A
* external SPI programmer

To SPI flash chip.

---

# 10. Extract BIOS Region

Now extract BIOS region from IFWI.

Why?
Because capsule update only updates BIOS region.

Command:

```bash
python BootloaderCorePkg/Tools/IfwiUtility.py extract \
  -i sbl_ifwi.bin \
  -p IFWI/BIOS \
  -o bios.bin
```

---

# 11. What Happens Here?

Extract:

```text
IFWI
 └── BIOS Region
      └── bios.bin
```

This gives:

```text
bios.bin
```

which contains:

* SBL firmware
* payload
* boot stages

But NOT:

* ME
* descriptor
* other flash regions

---

# 12. Why Capsule Uses BIOS Region Only

Runtime firmware update should NOT rewrite:

* ME firmware
* descriptor
* flash layout

So firmware update capsule only contains:

```text
bios.bin
```

---

# 13. Generate Capsule Update Image

Set keys:

```bash
export SBL_KEY_DIR=../SblKeys
```

Generate capsule:

```bash
python BootloaderCorePkg/Tools/GenCapsuleFirmware.py \
  -p BIOS bios.bin \
  -k KEY_ID_FIRMWAREUPDATE_RSA3072 \
  -o FwuImage.bin
```

---

# 14. What is Capsule Image?

`FwuImage.bin`

Contains:

* BIOS region
* metadata
* signature
* hash
* update header

Used for secure runtime firmware update.

---

# 15. Capsule Structure

Simplified:

```text
+----------------------+
| Capsule Header       |
+----------------------+
| Firmware Metadata    |
+----------------------+
| Signed BIOS Region   |
+----------------------+
| RSA Signature        |
+----------------------+
```

---

# 16. Firmware Update Flow

Copy:

```text
FwuImage.bin
```

to FAT USB drive.

Boot board.

Enter SBL shell.

Run:

```bash
fwupdate
```

---

# 17. Runtime Update Flow Internally

```text
fwupdate command
    ↓
SBL validates signature
    ↓
Verifies hash
    ↓
Enters firmware update mode
    ↓
Rewrites BIOS region
    ↓
Resets platform
    ↓
Completes update
```

---

# 18. Why Multiple Resets Happen

Firmware update may involve:

* staging
* partition swap
* verification
* redundant boot partitions

So multiple reboots are normal.

---

# 19. Overall Flow Diagram

```text
BuildLoader.py
      ↓
SlimBootloader.bin
      ↓
StitchLoader.py
      ↓
sbl_ifwi.bin
      ↓
IfwiUtility.py extract
      ↓
bios.bin
      ↓
GenCapsuleFirmware.py
      ↓
FwuImage.bin
      ↓
fwupdate
      ↓
SPI Flash Update
```

---

# 20. Key Difference Summary

| Component          | Meaning               | Contains               |
| ------------------ | --------------------- | ---------------------- |
| SlimBootloader.bin | SBL firmware only     | boot stages            |
| bios.bin           | BIOS region           | SBL + payload          |
| IFWI               | Full flash image      | descriptor + ME + BIOS |
| FwuImage.bin       | Signed update capsule | signed BIOS region     |

---

# 21. Typical Development Workflow

## Development

```text
build SBL
→ stitch IFWI
→ flash externally
```

---

## Field Updates

```text
generate capsule
→ fwupdate
→ runtime update
```

---

# 22. Useful Commands

## Build

```bash
python BuildLoader.py build adln50
```

---

## Stitch

```bash
python Platform/AlderlakeBoardPkg/Script/StitchLoader.py \
  -i factory_ifwi.bin \
  -s Outputs/adln50/SlimBootloader.bin \
  -o sbl_ifwi.bin \
  -p 0xAA000106
```

---

## Extract BIOS

```bash
python BootloaderCorePkg/Tools/IfwiUtility.py extract \
  -i sbl_ifwi.bin \
  -p IFWI/BIOS \
  -o bios.bin
```

---

## Set Keys

```bash
export SBL_KEY_DIR=../SblKeys
```

---

## Generate Capsule

```bash
python BootloaderCorePkg/Tools/GenCapsuleFirmware.py \
  -p BIOS bios.bin \
  -k KEY_ID_FIRMWAREUPDATE_RSA3072 \
  -o FwuImage.bin
```

---

## Trigger Update

Inside SBL shell:

```bash
fwupdate
```

```
```


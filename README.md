# Mobile Firmware & SoC Internals Handbook

An open-source, vendor-neutral engineering reference detailing modern mobile SoC bootchains, low-level partition layouts, Emergency Download Modes (EDL / BROM), Android Verified Boot (AVB 2.0), and hardware unbricking methodologies.

Maintained by **[Zeeshan Rajput (@Itszeeshanrajput)](https://github.com/Itszeeshanrajput)**.

---

## 📑 Table of Contents
1. [The Modern SoC Boot Chain](#1-the-modern-soc-boot-chain)
2. [Partition Architecture Reference](#2-partition-architecture-reference)
3. [Emergency Download Protocols (Deep Dive)](#3-emergency-download-protocols)
   - [Qualcomm Sahara & Firehose (EDL 9008)](#qualcomm-sahara--firehose-edl-9008)
   - [MediaTek BootROM (BROM) & Preloader](#mediatek-bootrom-brom--preloader)
   - [Samsung Exynos & Shannon Modem](#samsung-exynos--shannon-modem)
   - [Google Tensor Architecture](#google-tensor-gs-series)
4. [Android Verified Boot (AVB 2.0) & eFuses](#4-android-verified-boot-avb-20--efuses)
5. [Hardware Triage & Unbricking Playbook](#5-hardware-triage--unbricking-playbook)
6. [Glossary & Toolchains](#6-glossary--toolchains)

---

## 1. The Modern SoC Boot Chain

Every modern ARM64 mobile device boots through a strictly ordered chain of trust, transitioning from immutable silicon mask ROM up to the Linux kernel and user-space init.

```text
[Silicon Mask ROM (PBL / BROM)]
         │  (Hardware SHA verification via SoC Root of Trust eFuses)
         ▼
[Primary Bootloader / Preloader / XBL]
         │  (Initializes DDR memory, clocks, power PMIC)
         ▼
[TrustZone / ARM Trusted Firmware (EL3)] ◄───► [Secure OS / TEE / Keymaster]
         │
         ▼
[Secondary Bootloader / ABL / LK (EL2/EL1)]
         │  (Displays fastboot splash; parses AVB 2.0 vbmeta signatures)
         ▼
[Android Linux Kernel + Ramdisk (init_boot / boot)]
         │
         ▼
[User-Space Android Init (/system/bin/init)] ──► [Zygote] ──► [Android Runtime]
```

### Execution Privileges:
* **EL3 (Secure Monitor):** Highest privilege level; executes ARM Trusted Firmware (ATF). Controls cryptographic engines and hardware fuses.
* **EL2 (Hypervisor):** Optional virtualization layer (increasingly standard in Android 13+ with GKI and pKVM).
* **EL1 (Kernel/OS):** Linux kernel, device drivers, and core subsystem controllers.
* **EL0 (User-space):** Android framework, system daemons, native binaries, and applications.

---

## 2. Partition Architecture Reference

Since the introduction of **Dynamic Partitions (Android 10)**, **Generic Kernel Image (Android 12+)**, and **Virtual A/B (Android 11+)**, the storage layout on modern UFS/eMMC chips is divided into fixed partitions and a dynamic `super` container.

### Core Boot Partitions (Fixed Slices):
| Partition | Role & Composition |
| :--- | :--- |
| `boot` | GKI Linux kernel binary (`Image` or `Image.lz4`), DTB (Device Tree Blob) on legacy targets. |
| `init_boot` | Standardized in Android 13+. Houses the initial ramdisk (`init.rc`, first-stage init) independently of the kernel. |
| `vendor_boot` | Vendor-specific ramdisk containing kernel modules (`.ko`), device tree overlays, and `bootconfig`. |
| `dtbo` | Device Tree Blob Overlay; merged dynamically by the bootloader into the base device tree. |
| `vbmeta` | Cryptographic signature headers, public keys, and hash descriptors for Android Verified Boot 2.0. |
| `vbmeta_system` / `vbmeta_vendor` | Sub-chained verification trees for modular system and vendor images. |
| `super` | Logical container holding dynamic sub-partitions: `system`, `vendor`, `product`, `system_ext`, and `odm`. |
| `userdata` | Ext4/F2FS encrypted user data partition protected by Android Keystore. |

---

## 3. Emergency Download Protocols

When bootloader stages are corrupted or flash memory is erased, the SoC mask ROM falls back to an immutable, silicon-level recovery protocol over USB.

### Qualcomm Sahara & Firehose (EDL 9008)
* **USB Vendor/Product ID:** `05C6:9008` (Qualcomm HS-USB QDLoader 9008)
* **Phase 1: Sahara Protocol**
  - Handshake packet: `0x00000001` (Command Hello)
  - Exchange SoC HW_ID, OEM_ID, and PK_HASH.
  - Negotiates transmission of the signed Firehose programmer (`prog_firehose_lite.elf` or `.mbn`).
* **Phase 2: Firehose Protocol**
  - Once the ELF programmer runs in SoC SRAM, communication switches to XML-based streaming over USB.
  - Read/Write commands via XML payloads:
    ```xml
    <?xml version="1.0" ?>
    <data>
      <program SECTOR_SIZE_IN_BYTES="4096" num_partition_sectors="65536" 
               physical_partition_number="0" start_sector="0" 
               filename="boot.img" />
    </data>
    ```

### MediaTek BootROM (BROM) & Preloader
* **USB Vendor/Product ID:** `0E8D:0003` (BROM) / `0E8D:2000` (Preloader VCOM)
* **Handshake Sequence:**
  - Send start bytes `0xA0`, `0x0A`, `0x50`, `0x05` to initiate sync.
  - BROM responds with chip hardware code (e.g. `0x0817` for MT6877/Dimensity 900).
* **SLA & DAA (Authentication Bypass):**
  - MediaTek uses Serial Link Authentication (SLA) and Download Agent Authentication (DAA) to prevent unauthorized flashing.
  - Historical security research (e.g., *kamakiri*, *amonet*) leverages USB control transfer buffer overflows in BootROM to disable signature enforcement in memory.

### Samsung Exynos & Shannon Modem
* **Download Mode:** Custom framing over Samsung USB interface (`04E8:685D`).
* **PIT (Partition Information Table):** Binary lookup table parsed by `sboot` specifying start sectors, block counts, and partition names on UFS storage.

### Google Tensor (GS Series)
* Built in partnership with Samsung S.LSI, Tensor chips (GS101, GS201, G3) feature customized Shannon modem layers paired with the dedicated **Titan M2** security micro-controller.
* Flashing operations transition from the custom bootloader directly to **`fastbootd`** (userspace recovery) for dynamic partition management.

---

## 4. Android Verified Boot (AVB 2.0) & eFuses

Android Verified Boot guarantees the integrity of all bootable code from silicon to userspace:

```text
[Hardware eFuse: Root of Trust Public Key Hash]
         │ (Compares hash)
         ▼
[vbmeta Key Verification]
         │ (Signs vbmeta image)
         ▼
[Hash Tree Descriptors] ────────────────────────► [dm-verity]
         │                                            │
         ├─ Verifies boot / init_boot                 └─ Verifies runtime blocks
         └─ Verifies vendor_boot                         of /system and /vendor
```

### Rollback Index Protection:
* Security version numbers are burned into immutable silicon eFuses (anti-rollback fuses).
* If a firmware payload contains a rollback index lower than the value currently programmed in the SoC eFuse, the bootloader aborts execution immediately.

---

## 5. Hardware Triage & Unbricking Playbook

### DC Power Supply Current Draw Analysis (4.0V Benchmark):
* **`0.00 A` (No Draw):** Blown PMIC input protection diode, dead battery connection, or open circuit on VPH_PWR / VBAT.
* **`0.02 A – 0.05 A` (Static Low Draw):** SoC is stuck in Mask ROM mode (Qualcomm EDL 9008 or MediaTek BROM). Flash storage is either blank, corrupted, or missing clock/power rails.
* **`0.12 A – 0.25 A` (Looping Draw):** Primary bootloader (XBL/Preloader) loads into RAM, attempts to initialize DDR or UFS, encounters a CRC/cryptographic error, and resets.
* **`0.45 A – 0.90 A` (Hanging Draw):** Kernel panic during early boot; display backlight may be unpowered.

### Hardware Force-EDL Methods:
1. **EDL Test Points (TP):** Grounding the `FORCE_USB_BOOT` or `CMD` line resistor to ground during battery plug-in forces the BootROM to bypass internal storage scanning.
2. **Deep Flash Cable (Qualcomm 9008):** Shorting USB `D+` to `GND` via a momentary switch drops the data line impedance, signaling the SoC to enter hardware download mode.

---

## 6. Glossary & Toolchains

* **EDL:** Qualcomm Emergency Download Mode.
* **BROM:** MediaTek Boot Read-Only Memory.
* **SP Flash Tool:** MediaTek proprietary GUI for Preloader and BROM operations.
* **QPST / QFIL:** Qualcomm Product Support Tool / Qualcomm Flash Image Loader.
* **Fastbootd:** Userspace fastboot running inside recovery ramdisk for writing to dynamic partitions in `super`.
* **GKI:** Generic Kernel Image; standardized Google-built kernel architecture for Android 12+.

---

## Contributing & License
Contributions, documentation corrections, and hardware teardowns are welcome via Pull Requests.  
Licensed under the [MIT License](LICENSE).

# Lab 1B: Basic Computer Skills for Digital Forensics

**Course:** DFI101 / CIP-B101, Basic Computer Skills for Digital Forensics  
**Author:** Munirat Abdulfatah  
**Date:** 2 November 2026  
**Platform:** Windows 11 Pro (UEFI, GPT) on a physical laptop

This repository documents my work on Lab 1B. The lab covers how a computer stores data and boots, and how that knowledge applies to digital forensics. I examined my own machine, read its partition table, and verified partition sizes by hand using sector arithmetic.

---

## Lab objectives

- Capture basic system and storage information from a live Windows machine.
- Map evidence sources to system layers (applications, operating system, disk, memory, network).
- Explain the difference between a **program**, a **process** and a **thread**, and why process data matters in malware analysis and live triage.
- Calculate disk and partition sizes from sector counts.
- Inspect a partition table safely, without modifying the disk.
- Explain the boot process and state whether my system uses BIOS/Legacy or UEFI, backed by evidence.
- Verify the size of partitions 3 and 5 against the sizes reported by the operating system.

---

## Test system

| Item | Value |
|---|---|
| Operating system | Windows 11 Pro (Build 26300), x64 |
| Device | HP 250 G7 Notebook PC |
| Processor | Intel Core i5-1035G1 (4 cores, 8 logical processors) |
| Memory | 16 GB RAM |
| Firmware | Insyde F.48 (26/04/2024) |
| **BIOS mode** | **UEFI** |
| **Secure Boot** | **On** |
| Disk | SK hynix BC511 NVMe SSD, 476.94 GB, **GPT** |
| Logical / physical sector size | 512 bytes / 4096 bytes |

---

## Tools used

| Category | Tool / command | Purpose |
|---|---|---|
| Command line | Windows PowerShell | Disk and partition commands |
| Command line | Command Prompt (`systeminfo`) | System specification |
| PowerShell cmdlet | `Get-Disk` | Disk size, partition style, sector sizes |
| PowerShell cmdlet | `Get-Partition` | Partition numbers, types, offsets, sizes |
| PowerShell cmdlet | `Get-Volume` | File system type and OS-reported volume sizes |
| PowerShell cmdlet | `Get-Process` | Observe running processes |
| Windows utility | Disk Management (`diskmgmt.msc`) | Visual partition layout for comparison |
| Windows utility | System Information (`msinfo32`) | BIOS mode and Secure Boot state |
| Windows utility | Task Manager | Observe running processes |
| Other | Calculator, Snipping Tool, Microsoft Word | Checking arithmetic, screenshots, report writing |

---

## What I did, part by part

### Part A: System identification and evidence layer mapping

- **A1, system specification:** Ran `msinfo32` and `systeminfo` to capture the OS, firmware, BIOS mode, Secure Boot state and RAM.
- **A2, storage information:** Ran `Get-Disk`, `Get-Partition` and `Get-Volume` in PowerShell.
- **A3, evidence mapping:** Built a table following the guide *User activities = Applications + Operating System + Disk/Memory*.

| Layer | Example evidence |
|---|---|
| Applications | Browser history, cookies, downloads, email, chat logs, document edit history |
| Network | Firewall and router logs, DNS queries, proxy logs, packet captures |
| Operating system | Event logs, registry keys (UserAssist, RecentDocs, USB history), login records |
| Disk | File timestamps, deleted files, file slack, unallocated space, Recycle Bin |
| Memory / process | Running processes, open connections, loaded DLLs, encryption keys, clipboard |

**Why evidence exists in more than one layer:** a single action passes through several layers, and each layer records its own trace for its own purpose. Memory is volatile while disk data persists, so an item removed from one layer may survive in another. Matching traces across layers also corroborate each other and make tampering easier to detect.

### Part B: Program, process and thread investigation

- Observed running processes with **Task Manager** and `Get-Process | Select-Object -First 20 Name,Id,CPU,WorkingSet`.
- Explained the three concepts in my own words:

| Concept | Meaning | Analogy |
|---|---|---|
| Program | Instructions stored on disk, passive | A recipe in a cookbook |
| Process | A program that is running, with its own memory, PID and permissions | The act of cooking |
| Thread | A unit of work inside a process, sharing the process's memory | Individual cooks in one kitchen |

- Explained why process information helps in malware analysis and live triage: spotting lookalike or oddly placed processes, reading parent-child relationships (for example Word launching PowerShell), detecting injected code, tying processes to network connections, and capturing volatile evidence before shutdown.

### Part C: Disk sector and disk size calculation

PowerShell reports partition **offsets and sizes in bytes/MB/GB**, not sectors. I converted them using the 512-byte logical sector size.

```
Start sector      = Offset ÷ sector size
Number of sectors = Size in bytes ÷ sector size
Size in bytes     = Number of sectors × 512
Size in GB        = bytes ÷ 1024 ÷ 1024 ÷ 1024
```

The CHS formula (`Cylinders × Heads × Sectors per track × bytes per sector`) gives the same result when the standard logical geometry of 255 heads and 63 sectors per track is used. NVMe drives have no physical heads or cylinders, so LBA (sector count) is the reliable measure.

### Part D: Safe partition table inspection

Inspected the partition table using **read-only** tools (`diskmgmt.msc`, `Get-Disk`, `Get-Partition`, `Get-Volume`). Nothing was created, deleted, resized or formatted. I also confirmed the sector size with `Get-Disk | Select Number, LogicalSectorSize, PhysicalSectorSize, Size`.

### Part E: Boot process, MBR/GPT

Described the full boot sequence and confirmed my system boots in **UEFI** mode using three pieces of evidence:

1. `msinfo32` shows **BIOS Mode: UEFI**.
2. `Get-Disk` shows the partition style is **GPT**.
3. The disk has a 260 MB **EFI System Partition** and a 16 MB **Microsoft Reserved** partition, which is the standard layout for UEFI/GPT Windows installs.

### Part G: Final practical assignment

Verified the size of **partition 3** and **partition 5** from the partition table, recorded start sector, end sector and sector count, calculated the size in GB, and compared the results with the OS-reported sizes.

---

## Key results

### Partition table (Disk 0, GPT, 512-byte logical sectors)

| # | Type | Offset (bytes) | Start sector | Sectors | Size |
|---|---|---|---|---|---|
| 1 | System (EFI) | 1,048,576 | 2,048 | 532,480 | 260 MB (0.25 GB) |
| 2 | Reserved (MSR) | 273,678,336 | 534,528 | 32,768 | 16 MB |
| 3 | Basic (C:) | 290,455,552 | 567,296 | 791,513,088 | 377.42 GB |
| 4 | Recovery | 405,545,156,608 | 792,080,384 | 1,880,064 | 918 MB |
| 5 | Basic | 406,507,749,376 | 793,960,448 | 204,800,000 | 97.66 GB |

### Partition 3 (Windows C: drive)

| Step | Calculation | Result |
|---|---|---|
| Start sector | 290,455,552 ÷ 512 | **567,296** |
| Number of sectors | 792,080,384 (start of partition 4) − 567,296 | **791,513,088** |
| End sector | 567,296 + 791,513,088 − 1 | **792,080,383** |
| Bytes | 791,513,088 × 512 | 405,254,701,056 |
| KB, MB | ÷ 1024, ÷ 1024 | 395,756,544 KB, 386,481 MB |
| **Size in GB** | ÷ 1024 | **377.42 GB** |
| OS-reported | `Get-Volume` and Disk Management | 377.42 GB (NTFS) |

### Partition 5 (Basic data partition)

| Step | Calculation | Result |
|---|---|---|
| Start sector | 406,507,749,376 ÷ 512 | **793,960,448** |
| Number of sectors | 204,800,000 | **204,800,000** |
| End sector | 793,960,448 + 204,800,000 − 1 | **998,760,447** |
| Bytes | 204,800,000 × 512 | 104,857,600,000 |
| KB, MB | ÷ 1024, ÷ 1024 | 102,400,000 KB, 100,000 MB |
| **Size in GB** | ÷ 1024 | **97.66 GB** |
| OS-reported | `Get-Partition` and Disk Management | 97.66 GB |

### Cross-checks

- Partition 4 ends exactly where partition 5 starts: 792,080,384 + 1,880,064 = 793,960,448.
- The disk holds 512,110,190,592 bytes, which is 1,000,215,216 sectors. Space after partition 5 is 1,000,215,216 − 998,760,448 = 1,454,768 sectors, about 710 MB. Disk Management shows **710 MB unallocated** at the end of the disk, which supports the 204,800,000-sector figure for partition 5.
- Sector-based and CHS-based calculations agree (255 heads × 63 sectors per track = 16,065 sectors per cylinder).

### Why calculated and reported sizes match, or appear to differ

- Windows labels sizes "GB" but calculates them in binary units (1 GB = 1024³ bytes, strictly GiB). Using the same divisor gives matching results.
- Any difference of a few KB comes from rounding to two decimals.
- The drive advertised as 512 GB uses decimal units (512 × 10⁹ bytes), which Windows shows as 476.94 GB.
- The partitions do not add up to the full disk because of the GPT header and backup table, plus the unallocated space at the end.
- `Get-Volume` lists only partitions Windows has mounted as volumes. Partition 5 has no mounted volume, so Disk Management is the better comparison for it. Disk Management shows its file system as *Unknown*, which is typical of an encrypted (BitLocker) data partition.
- The disk reports a 512-byte logical sector on a 4096-byte physical sector, so it uses 512e emulation. The 512-byte logical size is the correct divisor for partition table values.

---

## Boot process

```mermaid


```

| Stage | What happens |
|---|---|
| 1. Power-on | Power reaches the motherboard, and the CPU runs its first instruction from the firmware chip. |
| 2. POST | The firmware checks CPU, RAM, storage and other essentials. |
| 3. BIOS/UEFI | Firmware initializes hardware. UEFI is the modern replacement for BIOS, with GPT, Secure Boot and large-disk support. |
| 4. CMOS/firmware settings | Saved settings (boot order, Secure Boot, date/time) are read from CMOS or NVRAM. |
| 5. Boot record | Legacy: MBR in the first sector. UEFI: GPT plus a boot file on the EFI System Partition (FAT32). |
| 6. Boot loader/manager | Windows Boot Manager (`bootmgfw.efi` on UEFI) or GRUB on Linux. |
| 7. Kernel loading | The kernel is loaded into memory with core drivers. |
| 8. OS control | The kernel starts services and the logon screen appears. |

| | Legacy BIOS | UEFI |
|---|---|---|
| Partition style | MBR | GPT |
| Boot record | First sector of disk | EFI System Partition |
| Secure Boot | No | Yes |
| Max disk size | About 2 TB | Very large |

**My system:** UEFI with GPT and Secure Boot on.

---

## Key takeaways

- Windows tools show **bytes and GB**, while forensic tools such as `mmls` and `fdisk` show **sectors**. Knowing the sector size lets you convert between them and calculate offsets (`start sector × 512`).
- Verifying a calculation against two independent sources (PowerShell and Disk Management) is a good habit in forensic work.
- Evidence rarely lives in one place. Correlating across layers gives stronger, more reliable findings.
- Process information is volatile and valuable, so it should be collected early in live triage.
- Always check the sector size instead of assuming 512 bytes, since many modern drives use 4096-byte sectors.

---

## Reproduce it yourself

Run PowerShell **as Administrator** on a Windows machine. All commands are read-only.

```powershell
# Sector size and disk details
Get-Disk | Select Number, FriendlyName, PartitionStyle, LogicalSectorSize, PhysicalSectorSize, Size

# Partition table with start, end and sector count calculated
Get-Partition -DiskNumber 0 | Select PartitionNumber, DriveLetter, Type, Offset, Size,
  @{n='StartSector';e={$_.Offset/512}},
  @{n='NumSectors';e={$_.Size/512}},
  @{n='EndSector';e={($_.Offset+$_.Size)/512-1}} | Format-Table -AutoSize

# OS-reported volume sizes and file systems
Get-Volume

# Confirm UEFI boot
msinfo32            # check "BIOS Mode" and "Secure Boot State"
```

Then compute `Size (GB) = NumSectors × 512 ÷ 1024³` and compare with Get-Volume and Disk Management (`diskmgmt.msc`).

> Adjust `512` if `LogicalSectorSize` on your disk reports a different value.

---

## Repository contents

```
.
├── README.md                              # This file
├── CIP-B101_Lab1B_Munirat_Abdulfatah.pdf  # Full lab report with screenshots
└── screenshots/                           # (optional) evidence images
```

---

## Disclaimer

This is an educational lab completed on my own computer. All commands were read-only, and no data was modified. Values such as serial numbers and device names have been omitted from this README.


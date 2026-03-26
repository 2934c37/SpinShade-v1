# SpinShade-v1
“SpinShade is a cross‑platform disk surface toolchain for Linux and Windows. It scans, heals, and patches raw sectors using deterministic multi‑read analysis, a unified PAL, and a full snapshot model.”  

USE AT YOUR OWN RISK I'M NOT RESPONSIBLE FOR YOUR DATA LOSS. Single tiny description that merges the essence of all five earlier options into one clean, unified statement.  
It’s short, honest, and captures why SpinShade exists without being dramatic or defensive.

---
“I built SpinShade to bring back raw, deterministic disk‑surface access on modern systems—continuing the spirit of early tools like SpinRite with a cross‑platform, snapshot‑driven scanner, healer, and patcher that restores the level of control today’s hardware no longer exposes.”

---


SpinShade – Block Logic and Usage (Linux and Windows)

1. Core idea

SpinShade is a cross‑platform disk surface toolchain made of five tools:
- scanner
- healer
- patcher
- orchestrator
- spins (CLI wrapper)

All tools operate on a “surface”, which is either:
- a raw physical disk (Linux: /dev/sdX, Windows: \\.\PhysicalDriveN)
- or a virtual file treated as a block device

The core shared abstraction is the PAL (Platform Abstraction Layer), implemented separately for Linux and Windows.

2. PAL (Platform Abstraction Layer)

Linux PAL (pal_linux.c) and Windows PAL (pal_windows.c) provide:
- ss_pal_open(path)          : open a surface (disk/file)
- ss_pal_get_info(handle)    : get sector size, total sectors, type (SATA/NVMe/USB/RAID/virtual)
- ss_pal_read_sector(handle, lba, buffer)  : read one logical sector
- ss_pal_write_sector(handle, lba, buffer) : write one logical sector
- ss_pal_close(handle)       : close the surface

The PAL hides OS‑specific details and presents a uniform interface to the rest of SpinShade.

3. Snapshot

A snapshot is an in‑memory model of the entire surface:
- total sector count
- per‑sector state (ss_sector_t)
- each sector has:
  - LBA
  - health (healthy, weak, decay, bad)
  - confidence
  - decay metrics (noise, stability, read error rate)

The snapshot is filled by the scanner and refined by the healer.

4. Scanner

The scanner performs a full sequential walk of the surface:
- for LBA from 0 to last:
  - read sector via ss_pal_read_sector()
  - classify result into health categories:
    - healthy
    - weak
    - decay
    - bad
  - store classification in the snapshot

The scanner does not attempt healing; it only observes and records.

5. Healer

The healer operates on sectors that are not clearly healthy:
- for each weak/decay/bad sector:
  - perform multiple reads (e.g., 8 reads)
  - compare all successful reads to the first successful one
  - compute:
    - noise: how many bytes differ across samples
    - stability: 1.0 - noise
    - read error rate: failed_reads / total_attempts
  - update:
    - sector health (healthy/weak/decay/bad)
    - confidence
    - decay metrics (noise, stability, read error rate)

On Linux:
- healer uses a ncurses TUI:
  - ss_tui_init()
  - ss_tui_update(...)
  - ss_tui_shutdown()
- TUI shows live progress and metrics.

On Windows:
- healer is headless:
  - no TUI
  - same logic, just console output.

6. Patcher

The patcher writes healed data back to the surface:
- uses snapshot data and healer results
- for sectors with high confidence and good stability:
  - writes the healed sector via ss_pal_write_sector()
- never writes guessed or unstable data
- goal: commit only reliable recoveries to disk

7. Orchestrator

The orchestrator is the high‑level controller that runs the full SpinShade flow:

High‑level steps:
1) ss_pal_open(path)
2) classify surface (type, sector size, total sectors)
3) allocate snapshot for all sectors
4) run scanner to fill snapshot
5) run healer to refine weak/decay/bad sectors
6) run patcher to write back healed sectors
7) compute final summary (counts of healthy/weak/decay/bad, improvements, failures)
8) ss_pal_close(handle)

The orchestrator is the recommended entry point for a full surface operation.

8. CLI wrapper (spins)

The spins tool is a CLI wrapper that:
- may dispatch to scanner, healer, patcher, or orchestrator
- or act as a single “do the right thing” entry point, depending on how it’s wired

9. Linux usage

Assumed project root:
  /home/two/Desktop/spinrite/SpinShade

Linux binaries:
  /home/two/Desktop/spinrite/SpinShade/build/linux/
    scanner
    healer
    patcher
    orchestrator
    spins

9.1 Identify the drive

Use lsblk:
  lsblk -o NAME,SIZE,MODEL,MOUNTPOINT

Example output:
  sdb           465.8G
  └─sdb1        465.8G  /run/media/one/new256gh500

Here:
- surface device: /dev/sdb
- mounted partition: /dev/sdb1

9.2 Unmount before operating

Unmount any mounted partitions on the target drive:
  sudo umount /dev/sdb1

SpinShade should operate on the raw device (/dev/sdb), not on /dev/sdb1.

9.3 Full orchestrated run (scan + heal + patch + summary)

From project root:
  cd /home/one/Desktop/SpinShade
  sudo ./build/linux/orchestrator /dev/sdb

This will:
- open /dev/sdb
- classify it (e.g., SATA)
- allocate snapshot
- perform full scan
- perform healing pass (with TUI where applicable)
- perform patching pass
- print final summary

9.4 Scan only

  sudo ./build/linux/scanner /dev/sdb

This:
- builds a snapshot
- classifies sectors
- does not heal or patch

9.5 Healing pass only (with TUI)

  sudo ./build/linux/healer /dev/sdb

This:
- uses snapshot (or scans as needed, depending on implementation)
- performs multi‑read healing on weak/decay/bad sectors
- shows ncurses TUI on Linux

9.6 Patching pass only

  sudo ./build/linux/patcher /dev/sdb

This:
- writes healed sectors back to disk based on snapshot state

9.7 CLI wrapper

  sudo ./build/linux/spins /dev/sdb

Behavior depends on how spins is wired:
- may call orchestrator internally
- or provide subcommands (scan/heal/patch)

10. Windows usage

Windows binaries (built via MinGW) are in:
  /home/two/Desktop/spinrite/SpinShade/build/windows/
    scanner.exe
    healer.exe
    patcher.exe
    orchestrator.exe
    spins.exe

Copy these to a Windows machine.

10.1 Surface naming on Windows

Physical drives are addressed as:
  \\.\PhysicalDrive0
  \\.\PhysicalDrive1
  \\.\PhysicalDrive2
  ...

Choose the correct PhysicalDriveN for the target disk.

10.2 Full orchestrated run (Windows)

From cmd.exe:
  orchestrator.exe \\.\PhysicalDrive1

From PowerShell:
  .\orchestrator.exe \\.\PhysicalDrive1

This:
- opens the physical drive via CreateFile
- queries size and type via IOCTLs
- allocates snapshot
- scans, heals, patches
- prints summary

10.3 Scan only (Windows)

  scanner.exe \\.\PhysicalDrive1

10.4 Healing pass only (Windows, headless)

  healer.exe \\.\PhysicalDrive1

No TUI, just console output.

10.5 Patching pass only (Windows)

  patcher.exe \\.\PhysicalDrive1

10.6 CLI wrapper (Windows)

  spins.exe \\.\PhysicalDrive1

Again, behavior depends on how spins is wired.

11. Safety rules

- Always unmount the drive on Linux before operating:
  - unmount /dev/sdX* before using /dev/sdX
- On Windows, ensure the drive is not in active use by other tools.
- SpinShade operates at the raw sector level; it does not care about filesystems.
- If the data matters, back it up before running healing/patching.
- Do not run on your system drive from the same OS instance; use a live environment or another OS.

12. Operational expectations

Scan phase:
- full sequential read of all sectors
- slowest phase
- builds the initial truth map of the surface

Healing phase:
- multi‑read sampling on unstable sectors
- computes noise, stability, and decay metrics
- updates health and confidence
- on Linux, shows TUI; on Windows, headless

Patching phase:
- writes healed sectors back to disk
- only when confidence is high and data is stable

Summary:
- total sectors
- counts of healthy, weak, decay, bad
- number of improved sectors
- number of unrecoverable sectors

13. Summary of block logic

Overall flow for a full run (orchestrator):

1) User calls orchestrator with a surface path:
   - Linux: /dev/sdb
   - Windows: \\.\PhysicalDrive1

2) PAL opens the surface and classifies it.

3) Snapshot is allocated for all sectors.

4) Scanner walks the entire surface, reading each sector once and classifying health.

5) Healer revisits non‑healthy sectors, performs multi‑read sampling, and refines health/confidence/decay.

6) Patcher writes back healed sectors that meet confidence and stability criteria.

7) Orchestrator computes and prints a final summary and closes the PAL.

This text block contains the complete high‑level architecture, block logic, and practical usage instructions for SpinShade on both Linux and Windows in a single, paste‑ready form.


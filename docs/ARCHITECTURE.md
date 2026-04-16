# BurnMonster Firmware Architecture Notes

This document captures architectural decisions and key implementation details for contributors and future maintenance.

## Counter persistence

The firmware uses a single global counter persisted in MCU flash at `0x0803FC00` (last 1 KB sector of the 256 KB flash). Implementation in `CartReaderApp/flashparam.c`:

- `save_dword(uint32_t)`: erases the sector and writes a new value (fixed in commit fb5323b — original firmware silently failed to erase, causing the counter to stick at 0)
- `load_dword()`: reads the persisted value at boot (`main.c:501`)

The counter is incremented on every successful write to the SD card, regardless of operation type:
- ROM dumps (`readROM_GB`, `readROM_GBA`, `readROM_GBM`)
- Save reads (`readSRAM_GB`, `readSRAM_GBA`, `readEeprom_GBA`, `readFLASH_GBA`)
- Mapping reads (NPower)

Folder structure: `<TYPE>/<OPERATION>/<NAME>/<COUNTER>/<NAME>.<ext>`

This means counter values are not consecutive within a single game's folder — gaps appear when other operations on other cartridges happen between saves of the same game.

## File browser

`fileBrowser()` in `CartReaderApp/Operate.c:305` lists directory contents on the device. Key constraints:
- Cache limited to 128 entries per directory
- 7 entries displayed per page (then break, paginate)
- Hidden files (starting with `.`) and `__MACOSX` directories are filtered out (added in commit 8a6ba56)

## Save slot management (Issue #2, in progress)

**Decision: Variant A** — extend the existing folder-based scheme.

### Folder structure
```
GB/SAVE/POKEMONY/1/POKEMONY.sav      ← auto-save (numeric, starting at 1)
GB/SAVE/POKEMONY/2/POKEMONY.sav
GB/SAVE/POKEMONY/-david/POKEMONY.sav ← user slot (- prefix, sorts above numerics)
GB/SAVE/POKEMONY/-partner/POKEMONY.sav
```

Sorting: ASCII `-` (0x2D) < `0` (0x30), so user slots always appear above auto-saves in the file browser.

### Counter strategy

Auto-save folders use a **per-game counter**, not the global MCU counter (`foldern`).

On each save operation, the firmware scans all numeric subfolders in `GB/SAVE/<GAMENAME>/`, finds the highest value, and uses that + 1 as the next slot number (minimum: 1). This makes the counter robust against firmware updates and SD card replacements, since the ground truth is always derived from what is actually on the SD card.

The global MCU counter (`foldern`) is preserved as-is and continues to be used exclusively for ROM dumps and other non-save operations.

### User-facing workflow (Read Save)
```
Read Save →
Does GB/SAVE/<GAMENAME>/ contain any slots?
├── No  → create new auto-save directly (next per-game slot number)
└── Yes → show slot selection menu:
    ├── [list of existing slots, user slots first]
    │     → select slot → confirm → overwrite
    │     → "Back" → return to this menu
    └── [New auto-save]
          → next per-game slot number → save
```

- User slots (`-name`) require double confirmation before overwrite
- Auto-saves require single confirmation before overwrite
- "Back" always returns to the slot selection menu, never to the main menu

### Manage Saves menu (new)

A new menu entry lists all slots for the currently inserted cartridge:
- Selecting a slot offers Delete with confirmation
- User slots: double confirmation required
- Auto-saves: single confirmation required

### Implementation order

1. GB/GBC first (`readSRAM_GB` in `GB.c:600`, `writeSRAM_GB` in `GB.c:672`)
2. GBA after GB is tested on hardware (three save types: SRAM, EEPROM, Flash)

### Key implementation hints

- `next_save_slot(const char *savePath)` — new helper in `Operate.c`/`Operate.h`: scans savePath, finds highest numeric subfolder, returns max+1 (min 1), skips `-` prefix entries
- `readSRAM_GB()` in `GB.c:600` — main entry point to modify
- `fileBrowser()` in `Operate.c:305` — reuse for slot selection UI
- Folder creation via FatFs `f_mkdir()`

## Time-independent naming (Issue #3)

The device has no RTC. Issue #3 is implicitly resolved by the global counter approach: filenames never depend on date/time.

## Hardware constraints

- MCU: GD32F103VC (Cortex-M3, 72 MHz, 256 KB flash, 96 KB RAM)
- Display: SSD1306 OLED 128×64 over I²C
- SD card: FAT32 + MBR partition table required
- Bootloader at `0x08000000–0x0800BFFF` (48 KB, source not provided by FunnyPlaying); application at `0x0800C000` onwards
- Recovery: place official `update.bin` at SD root, power on — bootloader auto-flashes

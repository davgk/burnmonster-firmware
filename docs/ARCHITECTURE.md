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

## Save slot management (planned, Issue #2)

**Decision: Variant A** — extend the existing folder-based scheme.

Current structure:
```
GB/SAVE/POKEMONY/0/POKEMONY.sav    ← auto-save (numeric subfolder)
GB/SAVE/POKEMONY/1/POKEMONY.sav    ← auto-save
```

Planned addition:
```
GB/SAVE/POKEMONY/-david/POKEMONY.sav     ← user slot (subfolder with - prefix)
GB/SAVE/POKEMONY/-partner/POKEMONY.sav   ← user slot
```

Sorting via the `-` prefix puts user slots above numeric auto-saves in the file browser (ASCII `-` < `0`).

Workflow on Read Save:
1. Check if `<NAME>` directory exists in `GB/SAVE/`
2. If existing slots/saves found, prompt user: Overwrite slot / Overwrite auto-save / Create new
3. Overwrite preserves the chosen folder name
4. Create new uses the next global counter value as folder name

Workflow on Manage Saves (new menu):
- List all `.sav` files for the inserted cart's `<NAME>` directory
- Selecting one offers Delete with confirmation
- User slots (folders starting with `-`) require double confirmation; auto-saves single confirmation

### Implementation hints
- `readSRAM_GB()` in `GB.c:600` is the main save-read entry point
- `writeSRAM_GB()` in `GB.c:672` for save-write
- `fileBrowser()` in `Operate.c:305` for slot selection UI (existing component, can be reused)
- Folder creation likely uses FatFs `f_mkdir()`; check existing usage in `GB.c` around the save path construction

## Time-independent naming (Issue #3)

The device has no RTC. Issue #3 is implicitly resolved by the global counter approach: filenames never depend on date/time.

## Hardware constraints

- MCU: GD32F103VC (Cortex-M3, 72 MHz, 256 KB flash, 96 KB RAM)
- Display: SSD1306 OLED 128×64 over I²C
- SD card: FAT32 + MBR partition table required
- Bootloader at `0x08000000–0x0800BFFF` (48 KB, source not provided by FunnyPlaying); application at `0x0800C000` onwards
- Recovery: place official `update.bin` at SD root, power on — bootloader auto-flashes

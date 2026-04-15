# BurnMonster Firmware

Community-maintained extended firmware for the **FunnyPlaying BurnMaster** portable cartridge flasher.

This project is an independent fork of [HDR/Burnmaster-Firmware](https://github.com/HDR/Burnmaster-Firmware) (which itself is the source provided by FunnyPlaying for firmware v1.10). It adds bug fixes and quality-of-life features that improve the daily usability of the device, while remaining fully compatible with the original hardware.

> ⚠️ **Unofficial:** This is not endorsed by FunnyPlaying. Use at your own risk. The official firmware can be flashed back at any time via the same `update.bin` mechanism.

## Status

This project is in active development. Current state: **v1.10 baseline + folder counter fix**.

## What this firmware does differently

### Bug fixes
- ✅ **Folder counter no longer stuck at 0.** The original firmware silently overwrote previous ROM dumps and save backups due to a missing flash erase in `save_dword()`. See [commit message](../../commits/main) for the technical detail.

### Planned features
See [open issues](../../issues) for the current roadmap. Contributions and feedback welcome.

## Compatibility

Hardware: FunnyPlaying BurnMaster (GD32F103VC, 256 KB flash, 1″ OLED, MicroSD).

Drop-in replacement for the official firmware. Bootloader is untouched, so you can always revert by flashing the official `update.bin` from FunnyPlaying.

## Building

This repository is set up for **automatic builds via GitHub Actions** on every push to `main` (or any branch matching `fix/**`). The compiled `update.bin` is available in the [Actions tab](../../actions) as a build artifact.

For local builds, see the original instructions: [SEGGER Embedded Studio 6.22a](https://www.segger.com/products/development-tools/embedded-studio/), open `CartReaderApp/GDCartReader.emProject`, build, rename `Output/Debug/Exe/GDCartReader.bin` to `update.bin`.

## Installation

1. Download the latest `update.bin` from the Actions tab (newest successful build).
2. Copy it to the root of an SD card formatted as FAT32 with MBR partition table.
3. Insert the SD card into the BurnMaster.
4. Power on. The bootloader will detect `update.bin` and flash it automatically.
5. After successful update, the file is deleted from the SD card.

## Reverting to official firmware

Download the official `update.bin` from FunnyPlaying and flash it the same way. The bootloader is in a separate flash region and is never touched by either firmware.

## Credits

- **Original firmware:** [FunnyPlaying](https://funnyplaying.com), provided as source via [HDR/Burnmaster-Firmware](https://github.com/HDR/Burnmaster-Firmware)
- **Underlying project:** [sanni/cartreader](https://github.com/sanni/cartreader) (the cartridge reader software the BurnMaster firmware is based on)
- **This fork:** [davgk](https://github.com/davgk), with diagnostic and patch assistance from Claude Code

## Disclaimer

This is **unofficial, community-maintained software** provided **as-is**, without any warranty of any kind, express or implied. Use is entirely at your own risk.

By flashing this firmware to your BurnMaster you acknowledge that:

- The author(s) of this project are **not affiliated with FunnyPlaying** and this firmware is not endorsed, supported, or distributed by them.
- The author(s) accept **no liability** for any damage to your hardware, loss of data (including ROM dumps, save files, or cartridge contents), bricked devices, or any other direct or indirect consequences of using this firmware.
- You are responsible for **maintaining your own backups** of any data on cartridges before performing read or write operations.
- The official FunnyPlaying firmware can be flashed back at any time via the same `update.bin` mechanism (see *Reverting to official firmware* above).
- If you encounter problems, please **contact the author(s) of this project, not FunnyPlaying** — they cannot help with modifications they did not produce.

The firmware is distributed under the GPL-3.0 license, which includes a formal disclaimer of warranty in its full text. This section is provided as a plain-language summary and does not replace the license terms.

## License

GPL-3.0, inherited from the original firmware.

# BurnMonster Firmware – Session Context Loader

> **Anleitung für mich (Claude):** Lies diese Datei beim Session-Start komplett durch, bevor du auf die erste Nachricht des Users antwortest. Sie enthält alles, was du wissen musst, um nahtlos weiterzumachen.

---

## Wer und was

**User:** David (Profil-Memory enthält DMF, Steinfurt-Borghorst, GBC-Hobby, etc.)

**Projekt:** BurnMonster Firmware – eine Custom-Firmware für den FunnyPlaying BurnMaster (portabler Game-Boy-Cart-Flasher).

**Repo:** https://github.com/davgk/burnmonster-firmware

**Lokaler Pfad auf Davids Mac:** `/Users/david/workspace/burnmonster-firmware`

**Archivierter alter Ordner:** `/Users/david/workspace/Burnmaster-Firmware-archived-2026-04-15` (kann nach 4 Wochen gelöscht werden)

---

## Workflow

David arbeitet mit **Claude Code** lokal auf dem Mac. Ich (Claude im Chat) schreibe die Prompts, die er dort reinkopiert. Claude Code führt Dateiänderungen, Commits und gh-Befehle aus. David gibt mir den Output zurück, ich bewerte und schreibe den nächsten Prompt.

**Wichtige Prompt-Regeln:**
- Immer „Show diff before applying" und „Wait for my confirmation" einbauen
- Nie sofort pushen – immer erst auf David warten
- `OledClear()` vor jedem `questionBox_OLED()` Call
- Sprache: Deutsch im Chat, Englisch in Code-Kommentaren und Commit-Messages

---

## Git-Konventionen

- Author und Committer immer: `davgk <226785137+davgk@users.noreply.github.com>`
- **Niemals** Co-Author-Zeilen für Claude
- **Niemals** Claude als Author oder Committer
- Build via GitHub Actions: automatisch bei Push auf `main`
- Build-Artifact: `Burnmonster-Firmware` (enthält `update.bin`, direkt flashbar)
- Recovery: Original `update.bin` von FunnyPlaying auf SD-Karte → Auto-Flash beim Boot

---

## Was bereits implementiert ist (Stand: 16. April 2026)

### Fixes
- **Save-Counter-Bug** (`flashparam.c`): fehlender Flash-Erase vor `save_dword()` — gefixt (commit fb5323b)
- **CI-Bug**: `upload-artifact@v3` → v4 (deprecated)
- **`foldern` Initialisierung**: startet jetzt bei 1 statt -1 (erased flash = 0xFFFFFFFF wurde als signed int -1 interpretiert) — Guard bei allen `load_dword()` Call Sites (commit 2654c45)
- **Workflow-Naming**: Artifact heißt `Burnmonster-Firmware`, Binary heißt `update.bin`

### Features
- **Issue #1 (Hidden Files Filter)**: Dateien mit `.`-Präfix und `__MACOSX`-Ordner werden im On-Device-Browser nicht angezeigt (commit 8a6ba56) — auf Hardware getestet ✓
- **Issue #2 (Save Slot Management, GB complete)**:
  - `next_save_slot()` in `Operate.c`/`Operate.h`: per-game Counter durch Scan der vorhandenen Unterordner
  - `readSRAM_GB()` in `GB.c`: Slot-Auswahl-Menü mit Sort, `.sav`-Validierung, OledClear zwischen States
  - `manageSaves_GB()` in `GB.c`: Delete mit Bestätigung, Loop-back nach Deletion
  - GB-Menü umstrukturiert: Read Save / Write Save / Manage Saves / Read Rom / Flash GBC Cart / NPower GB Memory / Reset
  - Auf Hardware getestet ✓

### Folder-Struktur (GB Saves)
```
GB/SAVE/POKEMONY/-david/POKEMONY.sav   ← user slot (- prefix, sorts first)
GB/SAVE/POKEMONY/1/POKEMONY.sav        ← auto-save (per-game counter, starts at 1)
GB/SAVE/POKEMONY/2/POKEMONY.sav
```
User-Slots werden intern mit `-` Prefix gespeichert aber sollen in Zukunft ohne `-` angezeigt werden (Issue #11).

---

## Offene Issues

| # | Titel | Priorität |
|---|---|---|
| #4 | Checksum verify uses wrong folder after multiple ROM dumps | Low |
| #5 | Write Save: replace fileBrowser with slot selection menu | Medium |
| #6 | Back-navigation: consistent Cancel button behavior across all screens | Medium |
| #7 | About screen: update branding to BurnMonster | Low |
| #8 | SD card info screen: show free space, card size and format | Low |
| #9 | All Saves: system-wide save browser with delete (GB and GBA) | Medium |
| #10 | All ROMs: system-wide ROM browser with delete (GB and GBA) | Low |
| #11 | UX polish: hide leading dash from user slot names in all menus | Low |

**Empfohlene nächste Schritte (in dieser Reihenfolge):**
1. Issue #11 (Bindestrich ausblenden) – minimal, sofort umsetzbar
2. Issue #7 (About-Screen) – 30 Minuten
3. Issue #5 (Write Save Slot-UI) – mittlerer Aufwand, baut auf readSRAM_GB-Pattern auf
4. Issue #9 (All Saves) – größeres Feature
5. GBA-Portierung von Issue #2 – alle GB-Pattern auf GBA übertragen

---

## Wichtige Code-Stellen

| Datei | Zeile (ca.) | Beschreibung |
|---|---|---|
| `CartReaderApp/flashparam.c` | — | Counter-Persistenz im MCU-Flash |
| `CartReaderApp/Common.c` | 15 | `int foldern` global declaration |
| `CartReaderApp/Operate.c` | ~690 | `next_save_slot()` |
| `CartReaderApp/Operate.c` | 305 | `fileBrowser()` |
| `CartReaderApp/GB.c` | ~609 | `readSRAM_GB()` Slot-Auswahl |
| `CartReaderApp/GB.c` | ~2304 | `manageSaves_GB()` |
| `CartReaderApp/GB.c` | ~2462 | `gbMenu()` |
| `CartReaderApp/GBA.c` | 642 | `readSRAM_GBA()` – noch nicht angepasst |
| `CartReaderApp/GBA.c` | 1117 | `readEeprom_GBA()` – noch nicht angepasst |
| `CartReaderApp/GBA.c` | 1418 | `readFLASH_GBA()` – noch nicht angepasst |

---

## Hardware-Constraints

- MCU: GD32F103VC (Cortex-M3, 72 MHz, 256 KB flash, 96 KB RAM)
- Display: SSD1306 OLED 128×64 über I²C — 7 Einträge pro Seite, `OledClear()` vor jedem Screen-Wechsel
- SD-Karte: FAT32 + MBR erforderlich
- Bootloader: `0x08000000–0x0800BFFF` (48 KB, nicht öffentlich); Anwendung ab `0x0800C000`
- Zwei Buttons + D-Pad — kein Touchscreen, keine Texteingabe möglich
- Alle Datums-/Zeitstempel auf SD-Karte zeigen 31.12.2021 23:00 (kein RTC)

---

## Bekannte latente Bugs (nicht prioritär)

- **Checksum verify** (Issue #4): nutzt `foldern - 1` um letzten ROM-Ordner zu finden — falsch wenn zwischen Dump und Verify eine andere Operation den Counter erhöht hat
- **Write Save** (Issue #5): nutzt `fileBrowser("/")` statt Slot-Auswahl-Menü
- **GBA Save-Pfad** (GB.c:2186): nutzt noch globalen Counter rückwärts statt per-game Scan

---

## Was du als Erstes tun solltest

1. Lies diese Datei komplett
2. Frag David wie er weitermachen will — wahrscheinlichste Optionen:
   - Issue #11 (Bindestrich ausblenden, 15 Minuten)
   - Issue #7 (About-Screen, 30 Minuten)
   - Issue #5 (Write Save Slot-UI, mittlerer Aufwand)
   - GBA-Portierung von Issue #2
3. **Nicht** sofort losbauen — erst Plan vorlegen, von David bestätigen lassen

---

## Tonfall

David ist technisch versiert, gründlich, schätzt ehrliche Einschätzungen. Bei Unsicherheit immer `[Nicht verifiziert]` verwenden. Keine Schmeichelei, keinen Aufwand untertreiben. Wenn etwas riskant ist: sagen.

---

**Stand dieses Dokuments:** Session vom 16. April 2026 — Issue #2 (GB) abgeschlossen und hardware-getestet.

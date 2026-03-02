# Menu System - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `menu.h` | 29 | Public API |
| `menu.cpp` | 7343 | Complete menu state machine |

---

## 1. Public API

```c
void HandleUI(void);              // Main menu state machine — called from scheduler
void OsdUpdate(void);             // Flush OSD to FPGA (from osd.h, called alongside)
void menu_key_set(uint32_t c);    // Inject key event into menu system
```

---

## 2. State Machine Architecture

The menu is a massive state machine with approximately **100 states** defined in `enum MENU`.

### State Naming Convention

States follow a **paired pattern**:
- `MENU_FOO1` — **Render** state: draws the menu page
- `MENU_FOO2` — **Input** state: handles user navigation/actions

The `parentstate` variable tracks which render state to return to after up/down navigation.

### Navigation Flow

```
MENU_NONE1/2          (idle, no OSD visible)
    |
    v (OSD button / F12 / menu key)
    |
MENU_8BIT_MAIN1/2    (core main menu — generic 8BIT)
    |--- Left/Right arrows --->  MENU_COMMON1/2 (system settings)
    |                                |--- Left/Right ---> MENU_MISC1/2 (misc settings)
    |                                |                         |--- Left/Right ---> back to MAIN
    v
MENU_FILE_SELECT1/2   (file browser — opened from menu items)
    |
    v
MENU_8BIT_MAIN1       (return to menu after file selection)
```

### Key State Groups

| State Prefix | Purpose |
|-------------|---------|
| `MENU_NONE1/2` | Idle — no OSD visible |
| `MENU_INFO` | Info overlay display |
| `MENU_8BIT_MAIN1/2` | Generic core main menu |
| `MENU_8BIT_SYSTEM1/2` | Core system/options page |
| `MENU_COMMON1/2` | System settings (shared across all cores) |
| `MENU_MISC1/2` | Miscellaneous settings (WiFi, BT, etc.) |
| `MENU_FILE_SELECT1/2` | File browser |
| `MENU_JOYSYSMAP` | Joystick system mapping |
| `MENU_BTPAIR` | Bluetooth pairing |
| `MENU_SAVE1/2` | Save state menu |
| `MENU_RESET1/2` | Reset confirmation |
| `MENU_MAIN1/2` | Minimig main menu |
| `MENU_SETTINGS*` | Minimig settings submenus |
| `MENU_ARCHIE_MAIN1/2` | Archie main menu |
| `MENU_ST_MAIN1/2` | Atari ST main menu |

---

## 3. Config String to Menu Mapping

The FPGA core provides a semicolon-delimited config string via `UIO_GET_STRING`. Each segment from index 2+ defines a menu entry.

### Config String Entry Prefixes

| Prefix | Syntax | Description |
|--------|--------|-------------|
| `F` | `F[S][index],EXT[&EXT..][;label]` | File load browser. `S` = full submenu, index = file slot |
| `S` | `S[index],EXT[&EXT..][;label]` | SD image mount (disk image) |
| `O` | `O[bits],val0,val1,...[;label]` | Option with status bit range. `[3:1]` or legacy `AB` format |
| `o` | Same as `O` | Option (extended status, +32 offset) |
| `T` | `T[bit][;label]` | Toggle (momentary pulse) |
| `t` | Same as `T` | Toggle (extended status) |
| `R` | `R[bit][;label]` | Reset + toggle (asserts reset, toggles bit, releases reset) |
| `r` | Same as `R` | Reset + toggle (extended) |
| `P` | `P[page_num][;label]` | Page/submenu navigation |
| `C` | `C[;label]` | Cheats menu |
| `-` | `-` | Separator line (non-selectable) |
| `DIP` | `DIP` | DIP switch settings |
| `CHEAT` | `CHEAT` | Cheat code entry |
| `H` | `H[bit]` | Hide following entry based on status bit |
| `D` | `D[bit]` | Disable (grey out) following entry based on status bit |
| `h` | `h[bit]` | Hide when bit is 0 (inverted) |
| `d` | `d[bit]` | Disable when bit is 0 (inverted) |
| `f` | `f[index],EXT` | Supplement file load (used for secondary files) |
| `J` | `J[swap],btn1,btn2,...` | Joystick button names. swap=1 enables joystick swap |
| `I` | `I,text` | Info strings (displayed as overlay) |
| `V` | `V,version` | Config version string |
| `X` | `X` | Disable OSD entirely |

### Status Bit Addressing

Options use bracket notation to specify which bits of the 128-bit status they control:
- `[3:1]` = bits 3 down to 1 (3 bits, 8 possible values)
- `[0]` = single bit 0 (2 values: off/on)
- Legacy format: `AB` where A-Z map to bit positions 0-25

For `o`/`t`/`r` (lowercase): bit positions are offset by +32 (extended status).

### Example Config String

```
"SNES;;FS0,SFC/SMC/BIN/BS;Load ROM;-;O[2:1],None,7MHz,8MHz,10MHz;CPU Speed;
T0,Reset;R0,Reset & Reload;P1,Audio Settings;S0,VHD;Mount SD Image;"
```

Parsed as:
- Index 0: `"SNES"` (core name)
- Index 1: `""` (empty special params)
- Index 2: `"FS0,SFC/SMC/BIN/BS"` -> File browser, slot 0, extensions .SFC/.SMC/.BIN/.BS
- Index 3: `"Load ROM"` -> Label for the file entry above (after semicolon)
- etc.

---

## 4. Menu Rendering

### `menumask` System

A **64-bit bitmask** tracks which menu rows are selectable:
- Bit `i` = 1: row `i` is selectable
- Up/down navigation skips unset bits and wraps around
- Non-selectable rows: separators, headers, info text

### Scrollable Menus

Large menus use a scroll mechanism:
1. `firstmenu` tracks the first visible row offset
2. `adjvisible` is set when the selected item scrolls off-screen
3. The render loop repeats with adjusted `firstmenu` until the selected item is visible

### Line Types

| Visual | Construction |
|--------|-------------|
| Normal text | `OsdWrite(line, text, 0, 0)` |
| Highlighted (selected) | `OsdWrite(line, text, 1, 0)` — inverted |
| Disabled/greyed | `OsdWrite(line, text, 0, 1)` — stippled |
| Separator | `OsdWrite(line, "", 0, 0)` — blank line |
| Page arrows | `OsdSetArrow(OSD_ARROW_LEFT \| OSD_ARROW_RIGHT)` |

---

## 5. File Browser

### Initialization

`SelectFile()` is called from menu action handlers:
```c
void SelectFile(const char *extensions, int mode, int active_dir,
                void (*callback)(const char *, unsigned char), char *path = NULL);
```

This sets up filesystem scanning variables and transitions to `MENU_FILE_SELECT1`.

### Features

- **Extension filtering:** Based on config string (e.g., `SFC/SMC/BIN`)
- **Type-to-search:** Keyboard characters filter the file list
- **Page navigation:** Page Up/Down for long listings
- **Recent files:** Grave/tilde key shows recently accessed files
- **Unmount:** Backspace unmounts the current image
- **Auto-enter:** Single-file directories auto-select (for MegaCD, PCE, etc.)
- **Zip support:** Shows ZIP files as directories; files inside ZIPs are selectable
- **Create-on-write:** Save files can be created when first written

### Modes

| Mode Flag | Effect |
|-----------|--------|
| `SCANF_INIT` | Initialize scan |
| `SCANF_END` | Finalize scan |
| `SCANF_NEXT` | Get next entry |
| `SCANF_PREV` | Get previous entry |
| `SCANF_NEXT_PAGE` | Jump forward by page |
| `SCANF_PREV_PAGE` | Jump back by page |
| `SCANO_DIR` | Include directories |
| `SCANO_UMOUNT` | Allow unmount option |
| `SCANO_CORES` | Scan for core files |
| `SCANO_TXT` | Text file mode |

---

## 6. Menu Action Flow

When the user selects a menu item:

### File Load (`F` prefix)

1. `SelectFile()` opens the file browser
2. User selects a file
3. Callback calls `user_io_file_tx()` which:
   - Sets file index (`FIO_FILE_INDEX`)
   - Sends extension info (`FIO_FILE_INFO`)
   - Starts download (`FIO_FILE_TX` with 0xFF)
   - Streams data in chunks (`FIO_FILE_TX_DAT`)
   - Ends download (`FIO_FILE_TX` with 0x00)

### SD Image Mount (`S` prefix)

1. `SelectFile()` opens the file browser
2. User selects an image file
3. Callback calls `user_io_file_mount()` which:
   - Opens the image file
   - Sends CSD/CID configuration (`UIO_SET_SDCONF`)
   - Sends image size (`UIO_SET_SDINFO`)
   - Sends mount notification (`UIO_SET_SDSTAT`)

### Option Change (`O`/`o` prefix)

1. User presses left/right to cycle through values
2. `user_io_status_set()` updates `cur_status[]` and sends 128-bit status to FPGA:
   ```
   spi_uio_cmd_cont(UIO_SET_STATUS2)
   // 8 x 16-bit words
   DisableIO()
   ```

### Toggle (`T`/`t` prefix)

1. User presses enter
2. Status bit is set to 1 momentarily, then cleared back to 0
3. Full 128-bit status sent each time

### Reset Toggle (`R`/`r` prefix)

1. User presses enter
2. Core reset asserted (`user_io_status_set("[0]", 1)`)
3. Status bit toggled
4. Core reset released

---

## 7. System Settings Pages

### Common Settings (MENU_COMMON1)

Available across all cores:
- Video processing (scaler filter, gamma, shadow mask)
- Audio filter selection
- Custom aspect ratio
- VSync adjust
- Video mode
- SNAC assignment
- Reset options

### Misc Settings (MENU_MISC1)

- WiFi setup
- Bluetooth pairing
- On-screen keyboard
- System information
- OSD lock configuration

---

## 8. MGL (Multi-Game Launcher)

MGL scripts automate menu navigation:

```xml
<mistergamedescription>
  <rbf>corename</rbf>
  <setname>gamename</setname>
  <file delay="1" type="f" index="0" path="path/to/rom"/>
  <file delay="1" type="s" index="0" path="path/to/disk.vhd"/>
  <reset delay="2"/>
</mistergamedescription>
```

MGL processing runs before user input, auto-navigating menus and selecting files according to the script with configurable delays.

---

## 9. Settings Persistence

### Generic Cores

- `user_io_status_save()` writes `cur_status[16]` (128-bit status) to `CORENAME.cfg`
- `FileSaveConfig(filename, data, size)` handles the file I/O
- Loaded on core init via `FileLoadConfig(filename, data, size)`
- Config version support: `CORENAME_v1.cfg`, `CORENAME_v2.cfg` (from `V` config string option)

### Minimig

Multi-slot config system in `minimig_config.cpp`:
- Up to 4 configuration slots
- Includes chipset, CPU, memory, video, floppy, HDD settings
- Saved to `Amiga/minimig.cfg`

### Atari ST

Separate config system in `st_tos.cpp`:
- TOS image selection
- Memory configuration
- Floppy/HDD settings

### Video/Audio Processing

Individual setter functions persist their settings:
- Scaler filter coefficients
- Gamma curves
- Shadow masks
- Audio filters
- Aspect ratio overrides

### UART

UART settings saved via `FileSaveConfig()` to `CORENAME_uart.cfg`.

---

## 10. OSD Lock System

Configurable directional code entry:
- Sequence of U/D/L/R/A/B inputs (up to 14 steps)
- Auto-lock timeout
- Configured via Misc settings page
- Prevents accidental OSD activation

---

## 11. Key Handling

### Input Sources

Menu keys come from:
- Physical OSD/USR buttons (via `fpga_get_buttons()`)
- Keyboard (F12/Menu key opens OSD, arrow keys navigate)
- Joystick (D-pad for navigation, A/B for select/back)
- `menu_key_set()` for injected keys

### Key Constants

Standard menu navigation uses bitmask keys:
- Up, Down, Left, Right
- Enter (select/confirm)
- Back (ESC/close)
- Page Up, Page Down

### Key Repeat

- Initial delay: 500ms (`REPEATDELAY`)
- Repeat rate: 50ms (`REPEATRATE`)

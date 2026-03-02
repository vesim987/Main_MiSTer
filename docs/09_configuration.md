# Configuration System - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `cfg.h` | ~200 | `cfg_t` structure definition |
| `cfg.cpp` | 794 | INI parser implementation |

---

## 1. INI File Format

The primary configuration file is `MiSTer.ini` located on the SD card root.

### Section Types

| Section | Format | Description |
|---------|--------|-------------|
| Global | `[MiSTer]` | Default settings for all cores |
| Per-core | `[corename]` | Override settings for a specific core (e.g., `[SNES]`, `[Genesis]`) |
| Video mode | `[video=mode]` | Settings for a specific video mode |
| Wildcard | Supports `*` and `?` | Match multiple core names |

### Section Priority

Settings are applied in order of specificity:
1. Global `[MiSTer]` section (lowest priority)
2. Per-core `[corename]` section (overrides global)
3. Video mode `[video=mode]` section (overrides per-core)

### Alternate INI Files

Up to 3 alternate INI configurations supported:
- `MiSTer.ini` (primary)
- `MiSTer_alt1.ini`
- `MiSTer_alt2.ini`
- `MiSTer_alt3.ini`

Switched via `altcfg()` in shared memory and `user_io_set_ini()`.

---

## 2. `cfg_t` Structure — All Fields

### Video Settings

```c
int video_mode;             // Default video mode (0-15)
int video_mode_def;         // Default video mode for current core
int video_mode_adj;         // Adjusted video mode
int vsync_adjust;           // VSync adjustment (0=none, 1=adjust, 2=low latency)
int vscale_mode;            // Vertical scaling mode (0-3)
int vscale_border;          // Vertical border size (pixels)
int vga_scaler;             // VGA scaler enable (0/1)
int vga_sog;                // VGA Sync on Green (0/1)
int forced_scandoubler;     // Force scandoubler (0/1)
int ypbpr;                  // YPbPr mode (0/1)
int composite_sync;         // Composite sync (csync) (0/1)
int direct_video;           // Direct video output (0/1)
int hdmi_limited;           // HDMI limited range (0=full, 1=limited, 2=limited2)
int dvi_mode;               // DVI mode - no audio over HDMI (0/1)
int hdr;                    // HDR mode (0/1/2)
int video_info;             // Video info display time (seconds, 0=off)
int osd_rotate;             // OSD rotation (0=none, 1=CW, 2=CCW)
int osd_timeout;            // OSD auto-hide timeout (seconds)
int controller_info;        // Controller info display time
int refresh_min;            // Minimum refresh rate
int refresh_max;            // Maximum refresh rate
```

### Scaling and Filters

```c
int vfilter_default;        // Default vertical filter
int vfilter_scanlines;      // Scanline filter
int sfilter_default;        // Default scaler filter
int afilter_default;        // Default audio filter
int custom_aspect_ratio[2][2]; // Custom AR: [0]=horizontal [1]=vertical, [x][0]=num [x][1]=den
```

### Audio

```c
int volumectl;              // Volume control level
int audio_96k;              // 96kHz audio mode (0/1)
```

### System

```c
char main[256];             // Alternate main executable path
int bootcore_timeout;       // Boot core auto-load timeout (seconds)
int fb_terminal;            // Framebuffer terminal mode (0/1)
int fb_size;                // Framebuffer size
int log_file_entry;         // Log file entries (0/1)
int rumble;                 // Rumble feedback enable (0/1)
int wheel_force;            // Wheel force feedback
```

### Input

```c
int mouse_throttle;         // Mouse movement speed scaling
int kbd_nomouse;            // Disable keyboard mouse emulation (0/1)
int sniper_mode;            // Mouse sniper/precision mode (0/1)
int spinner_throttle;       // Spinner speed scaling
int spinner_axis;           // Spinner axis (0=X, 1=Y)
int browse_expand;          // File browser expanded view
int gamepad_defaults;       // Use default gamepad mappings
int key_menu_as_rgui;       // Map Menu key as Right GUI
int keyrah_mode;            // Keyrah keyboard mode
int reset_combo;            // Reset key combo (0=Ctrl+Alt+RAlt, 1=Ctrl+LGui+RGui, 2=Ctrl+Alt+Del)
```

### Network

```c
int waitmount;              // Wait for network mount at boot
char shared_folder[256];    // Network share path
```

### Bluetooth

```c
int bt_auto_disconnect;     // Bluetooth auto-disconnect timeout
int bt_reset_before_pair;   // Reset BT adapter before pairing
```

---

## 3. INI Parser (`cfg_parse()`)

```c
void cfg_parse(int active = 0);
```

### Parsing Algorithm

1. Open `MiSTer.ini` (or alternate INI file)
2. Initialize `cfg` to defaults
3. For each line:
   - Skip comments (`#` or `;` prefix)
   - Detect section headers (`[...]`)
   - Parse `key=value` pairs
4. Apply matching sections:
   - `[MiSTer]` always applies
   - `[corename]` applies if it matches the current core name (case-insensitive)
   - Wildcard `[core*]` patterns are matched
5. Video mode sections applied after core sections

### Key=Value Parsing

Values are parsed as:
- **Integer:** decimal or hex (with `0x` prefix)
- **String:** unquoted text
- **Boolean:** 0/1 or yes/no

### INI Key Names

Complete mapping of INI key names to `cfg_t` fields:

| INI Key | Field | Type | Default |
|---------|-------|------|---------|
| `video_mode` | `video_mode` | int | 0 |
| `vsync_adjust` | `vsync_adjust` | int | 0 |
| `vscale_mode` | `vscale_mode` | int | 0 |
| `vscale_border` | `vscale_border` | int | 0 |
| `vga_scaler` | `vga_scaler` | int | 0 |
| `vga_sog` | `vga_sog` | int | 0 |
| `forced_scandoubler` | `forced_scandoubler` | int | 0 |
| `ypbpr` | `ypbpr` | int | 0 |
| `composite_sync` | `composite_sync` | int | 0 |
| `direct_video` | `direct_video` | int | 0 |
| `hdmi_limited` | `hdmi_limited` | int | 0 |
| `dvi_mode` | `dvi_mode` | int | 0 |
| `hdr` | `hdr` | int | 0 |
| `video_info` | `video_info` | int | 0 |
| `osd_rotate` | `osd_rotate` | int | 0 |
| `osd_timeout` | `osd_timeout` | int | 0 |
| `volumectl` | `volumectl` | int | 0 |
| `audio_96k` | `audio_96k` | int | 0 |
| `mouse_throttle` | `mouse_throttle` | int | 0 |
| `kbd_nomouse` | `kbd_nomouse` | int | 0 |
| `sniper_mode` | `sniper_mode` | int | 0 |
| `spinner_throttle` | `spinner_throttle` | int | 0 |
| `spinner_axis` | `spinner_axis` | int | 0 |
| `reset_combo` | `reset_combo` | int | 0 |
| `key_menu_as_rgui` | `key_menu_as_rgui` | int | 0 |
| `bootcore` | `bootcore` | string | "" |
| `bootcore_timeout` | `bootcore_timeout` | int | 0 |
| `fb_terminal` | `fb_terminal` | int | 0 |
| `fb_size` | `fb_size` | int | 0 |
| `rumble` | `rumble` | int | 1 |
| `wheel_force` | `wheel_force` | int | 50 |
| `shared_folder` | `shared_folder` | string | "" |
| `waitmount` | `waitmount` | int | 0 |
| `bt_auto_disconnect` | `bt_auto_disconnect` | int | 0 |
| `bt_reset_before_pair` | `bt_reset_before_pair` | int | 0 |
| `browse_expand` | `browse_expand` | int | 0 |
| `custom_aspect_ratio_1` | `custom_aspect_ratio[0]` | string | "" |
| `custom_aspect_ratio_2` | `custom_aspect_ratio[1]` | string | "" |
| `main` | `main` | string | "" |
| `vfilter_default` | `vfilter_default` | string | "" |
| `sfilter_default` | `sfilter_default` | string | "" |
| `afilter_default` | `afilter_default` | string | "" |

---

## 4. Core Status Configuration

### 128-bit Status

Each core has a 128-bit (16-byte) status word stored in `cur_status[16]`. Individual menu options map to bit ranges within this status.

### Status Persistence

```c
// Save status to file
int user_io_status_save(const char *filename);
    // Writes cur_status[16] to config/CORENAME.cfg (or CORENAME_v1.cfg)

// Load status from file
int FileLoadConfig(const char *filename, void *data, int size);
    // Reads up to 16 bytes from config/CORENAME.cfg into cur_status
```

### Config Version

Cores can declare a config version via the `V` option in the config string:
```
V,v1
```

This causes the config file to be named `CORENAME_v1.cfg` instead of `CORENAME.cfg`, allowing cores to break compatibility with old configs.

---

## 5. SDRAM Configuration

Stored in shared memory at `0x1FFFF000 + 0xF00`:

```c
uint16_t sdram_sz(int sz = -1);
```

- `sz = -1`: Read current SDRAM config
- `sz >= 0`: Write SDRAM config

Format in memory:
```
Offset 0: 0x12 (magic byte 1)
Offset 1: 0x57 (magic byte 2)
Offset 2-3: 16-bit SDRAM size/type value
```

---

## 6. Alternate Configuration Index

Stored in shared memory at `0x1FFFF000 + 0xF04`:

```c
uint16_t altcfg(int alt = -1);
```

- `alt = -1`: Read current alt config index
- `alt >= 0`: Write alt config index

Format in memory:
```
Offset 0: 0x34 (magic byte 1)
Offset 1: 0x99 (magic byte 2)
Offset 2: 0xBA (magic byte 3)
Offset 3-4: 16-bit alt config index
```

Used to switch between `MiSTer.ini`, `MiSTer_alt1.ini`, `MiSTer_alt2.ini`, `MiSTer_alt3.ini`.

---

## 7. UART Configuration

```c
int GetUARTMode();
void SetUARTMode(int mode);
uint32_t GetUARTbaud(int mode);

// Sent to FPGA via:
spi_uio_cmd_cont(UIO_SET_UART);   // 0x3B
spi_w(mode);
spi_w(baud & 0xFFFF);
spi_w(baud >> 16);
DisableIO();
```

Modes 1-5 (4 and 5 are remapped to 1). Baud rates from core config string `U` parameter.

Persisted to `CORENAME_uart.cfg`:
```c
FileSaveConfig("CORENAME_uart.cfg", &uart_mode, sizeof(uint32_t));
```

---

## 8. Video Processing Configuration

Various video settings are sent to the FPGA via dedicated SPI commands:

| Setting | SPI Command | Persistence |
|---------|-------------|-------------|
| Scaler filter | `UIO_SET_FLTNUM` (0x2B) | INI `sfilter_default` |
| Filter coefficients | `UIO_SET_FLTCOEF` (0x2A) | Filter file |
| Gamma enable | `UIO_SET_GAMMA` (0x32) | Per-core in INI |
| Gamma curve | `UIO_SET_GAMCURV` (0x33) | Gamma file |
| Audio filter | `UIO_SET_AFILTER` (0x39) | INI `afilter_default` |
| Shadow mask | `UIO_SHADOWMASK` (0x3E) | Shadow mask file |
| Custom aspect ratio | `UIO_SET_AR_CUST` (0x3A) | INI `custom_aspect_ratio` |
| Video mode | `UIO_SET_VIDEO` (0x20) | INI `video_mode` |
| Max height | `UIO_SETHEIGHT` (0x27) | Calculated |
| Max width | `UIO_SETWIDTH` (0x37) | Calculated |
| Volume | `UIO_AUDVOL` (0x26) | INI `volumectl` |
| Y/C parameters | `UIO_SET_YC_PAR` (0x41) | yc.txt file |

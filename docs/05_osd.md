# OSD (On-Screen Display) - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `osd.h` | 46 | Public API header |
| `osd.cpp` | 682 | OSD implementation |
| `charrom.h` | 8 | Font declaration |
| `charrom.cpp` | 217 | Font data + custom font loader |
| `logo.h` | 175 | MiSTer logo bitmap data |

---

## 1. OSD SPI Command Bytes

These are the fundamental command opcodes sent to the FPGA over the OSD chip-select:

```c
#define OSD_CMD_WRITE    0x20      // Write pixel data to OSD line
#define OSD_CMD_ENABLE   0x41      // Enable OSD
#define OSD_CMD_DISABLE  0x40      // Disable OSD
```

### Command Byte Encoding

| Byte Sent | Meaning |
|-----------|---------|
| `0x20 \| line` | Write 256 bytes of data to OSD line `line` (0-31, 5 bits) |
| `0x40` | Disable OSD |
| `0x40` + 5 words | Set OSD rotation (extended disable) |
| `0x41` | Enable OSD (plain menu mode) |
| `0x41 \| 0x02` | Enable OSD + disable keyboard passthrough |
| `0x41 \| 0x04` | Enable OSD in info mode (followed by x, y, width, height words) |
| `0x41 \| 0x08` | Enable OSD in message window mode |
| `0x41 \| 0x0A` | Enable OSD + disable keyboard + message mode |

### Mode Flag Constants

```c
#define DISABLE_KEYBOARD  0x02    // Suppress keyboard passthrough while OSD active
#define OSD_INFO          0x04    // Info overlay mode
#define OSD_MSG           0x08    // Message window mode
```

---

## 2. OSD Memory Layout

### Frame Buffer

```c
static uint8_t osdbuf[256 * 32];     // 8192 bytes total
```

- **256 bytes per line**, up to **32 lines** addressable
- Default `osd_size = 8` lines; can be set to 16 for larger menus
- When `is_menu()` is true (MENU core), `OsdUpdate()` sends 19 lines

### Dirty-Line Tracking

```c
static int osdset = 0;               // 32-bit bitmask: bit i = 1 means line i is dirty
```

Setting `osdset = -1` marks all 32 lines dirty.

### Per-Line Layout (256 bytes)

```
Bytes [0..2]    : 3 x 0xFF (white left border -- sidestripe)
Bytes [3..18]   : 8 rotated character columns x 2 (doubled) = 16 bytes (vertical title)
Byte  [19]      : 0xFF (white right border -- sidestripe)
Bytes [20..21]  : 2 x 0x00 (blue gap separator)
Bytes [22..255] : 234 bytes text content area (~29 characters at 8 px/char)
```

### Pixel Format

Each byte represents a **vertical column of 8 pixels**:
- Bit 0 = top pixel
- Bit 7 = bottom pixel
- Column-major, LSB-top format

### Starfield Overlay

```c
char framebuffer[16][256];            // 16 x 256 overlay buffer for star animation
```

OR'd into OSD output when `usebg=1`.

---

## 3. Font System

### Font Data

```c
extern unsigned char charfont[256][8]; // 256 characters, each 8 bytes (8x8 pixels, column-major)
```

### Character Map

| Range | Description |
|-------|-------------|
| 0x00 | Null/blank |
| 0x01-0x03 | Dither patterns (0x55, 0x2A, 0x14 alternating) |
| 0x04 | Bluetooth icon |
| 0x0E-0x0F | Atari logo halves |
| 0x10 | Arrow left (big, solid triangle) |
| 0x11 | Arrow right (big, solid triangle) |
| 0x12 | Arrow up |
| 0x13 | Arrow down |
| 0x14 | Bar top half |
| 0x15 | Bar bottom half |
| 0x16 | Mini arrow right |
| 0x17 | Write protect icon |
| 0x18 | Write enable icon |
| 0x19 | Unchecked checkbox |
| 0x1A | Checked checkbox |
| 0x1B | Middle dot |
| 0x1C | Ethernet icon |
| 0x1D | WiFi icon |
| 0x1E | Charge/lightning icon |
| 0x1F | Battery icon |
| 0x20-0x7E | Standard ASCII (space through tilde) |
| 0x7F | Solid block (all 0x7F) |
| 0x80-0x85 | Dotted frame: TL, T/B, TR, L/R, BR, BL |
| 0x86-0x8B | Solid frame: TL, T/B, TR, L/R, BR, BL |
| 0x8C | Empty square |
| 0x8D | Speaker icon |
| 0x8E-0x91 | Fill bar levels 1-4 |
| 0x92 | Memory slot empty |
| 0x93 | Memory 32MB indicator |
| 0x94 | Memory 64MB indicator |
| 0x95 | Memory 128MB indicator |
| 0x96 | Checkmark |

### Frame Characters

Used by `OSD_PrintInfo()` with frame parameter:
- `frame=1` -> dotted: chars 0x80-0x85 (base offset 0)
- `frame=2` -> solid: chars 0x86-0x8B (base offset 6)

Within each frame set:
- `+0` = Top-Left corner
- `+1` = Top/Bottom horizontal edge
- `+2` = Top-Right corner
- `+3` = Left/Right vertical edge
- `+4` = Bottom-Right corner
- `+5` = Bottom-Left corner

### Custom Font Loading

```c
void LoadFont(char* name)
```

Loads external font file (up to 2048 bytes):
- 768 bytes: starts at char 32, covers 96 chars
- Otherwise: auto-detect offset (chars 0-31 all zero = start at 32)
- Input format: row-major, MSB-left
- Transposes to column-major format used by `charfont[][]`

---

## 4. All Rendering Functions

### `OsdSetSize(int n)`

Sets number of OSD lines (8 or 16).

### `OsdGetSize()` -> `int`

Returns current OSD line count.

### `OsdSetTitle(const char *s, int a = 0)`

Renders title string into `titlebuffer[256]`:
1. Renders characters condensing zero-column gaps (spaces get up to 5 zero cols)
2. Centers content vertically within available height
3. Rotates in 8-byte chunks via `rotatechar()` for vertical display in sidestripe
4. Stores arrow flags: `OSD_ARROW_LEFT` (1) and `OSD_ARROW_RIGHT` (2)

### `OsdSetArrow(int a)`

Updates arrow flags without re-rendering the title.

### `OsdWrite(unsigned char n, const char *s, unsigned char invert, unsigned char stipple, char usebg, int maxinv, int mininv)`

Wrapper for `OsdWriteOffset(n, s, invert, stipple, 0, 0, usebg, maxinv, mininv)`.

### `OsdWriteOffset(unsigned char n, const char *s, unsigned char invert, unsigned char stipple, char offset, char leftchar, char usebg, int maxinv, int mininv)`

**Primary text rendering function.** Parameters:

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `n` | `unsigned char` | — | OSD line number (0 to osd_size-1) |
| `s` | `const char *` | `""` | Text string to display |
| `invert` | `unsigned char` | 0 | XOR all output with 0xFF for highlighting |
| `stipple` | `unsigned char` | 0 | Apply dither pattern (dimming for disabled items) |
| `offset` | `char` | 0 | Vertical pixel shift for sub-pixel scrolling |
| `leftchar` | `char` | 0 | Override sidestripe with this char (rotated) |
| `usebg` | `char` | 0 | OR starfield framebuffer into output |
| `maxinv` | `int` | 32 | Character position where inversion stops |
| `mininv` | `int` | 0 | Character position where inversion starts |

**Rendering pipeline:**

1. Calculate line limit (256, or 256-22 if right arrow on last line)
2. Render sidestripe (22 bytes) via `draw_title()`
3. If last line + left arrow: render arrow chars 0x10+0x14 (24 bytes)
4. For each character byte:
   - `0x0B`: Toggle stipple
   - `0x0C`: Toggle per-character XOR
   - `0x0D`/`0x0A`: Newline (advance to next OSD line)
   - Normal: render 8 font bytes with offset, XOR, stipple, optional BG overlay
5. Pad remaining bytes
6. If last line + right arrow: render arrow chars 0x15+0x11 (22 bytes)

### `OsdShiftDown(unsigned char n)`

Shifts all pixels in the text area of line `n` down by 1 pixel (left-shift each byte). Sidestripe untouched.

### `OsdDrawLogo(int row)`

Renders one row of the MiSTer logo:
1. Sidestripe
2. Logo data from `logodata[row][]` (227 bytes, 10 rows total)
3. OR with starfield `framebuffer[row][]`

### `OSD_PrintInfo(const char *message, int *width, int *height, int frame)`

Renders an info overlay:
1. Parses message into 32x16 character grid (handles `\n`, ignores `\r`)
2. Auto-calculates dimensions if width/height are 0
3. Optional frame: dotted (frame=1) or solid (frame=2) box-drawing chars
4. Renders characters starting at byte 0 (no sidestripe)

### `OsdClear(void)`

Zeroes first 16 lines (4096 bytes) of `osdbuf`. Marks all lines dirty.

### `StarsInit()`

Initializes 64 stars with:
- X: random 0-227 (fixed-point <<4)
- Y: random 0-127 (fixed-point <<4)
- dx: random -3 to -10 (leftward)
- dy: 0

### `StarsUpdate()`

Updates starfield: clears framebuffer, moves stars, respawns off-screen stars at right edge, plots into framebuffer. Marks all lines dirty.

### `ScrollText(char n, const char *str, int off, int len, int max_len, unsigned char invert, int idx)`

Smooth horizontal scrolling text:
1. Only advances on timer tick (initial 1000ms delay, then 10ms per step)
2. Splits string into non-scrolling header (first `off` chars) and scrollable portion
3. 1-pixel-per-step scrolling with 10-space gap between repetitions
4. Wraps at `(len + BLANKSPACE) * 8` pixels
5. 2 independent scroll contexts (idx 0 or 1)

### `ScrollReset(int idx = 0)`

Resets scroll position to 0 and timer to 1000ms delay.

---

## 5. OSD Enable/Disable Protocol

### `OsdEnable(unsigned char mode)`

```c
user_io_osd_key_enable(mode & DISABLE_KEYBOARD);
mode &= (DISABLE_KEYBOARD | OSD_MSG);
spi_osd_cmd(OSD_CMD_ENABLE | mode);    // sends 0x41 | mode
```

### `InfoEnable(int x, int y, int width, int height)`

```c
user_io_osd_key_enable(0);
spi_osd_cmd_cont(OSD_CMD_ENABLE | OSD_INFO);   // sends 0x45
spi_w(x);        // screen X position
spi_w(y);        // screen Y position
spi_w(width);    // width in characters
spi_w(height);   // height in characters
DisableOsd();
```

### `OsdDisable()`

```c
user_io_osd_key_enable(0);
spi_osd_cmd(OSD_CMD_DISABLE);     // sends 0x40
```

### `OsdRotation(uint8_t rotate)`

```c
spi_osd_cmd_cont(OSD_CMD_DISABLE);  // sends 0x40
spi_w(0); spi_w(0); spi_w(0); spi_w(0);  // 4 padding words
spi_w(rotate);                             // rotation value
DisableOsd();
```

Rotation values:
- `cfg.osd_rotate == 1` -> sends `3`
- `cfg.osd_rotate == 2` -> sends `1`
- otherwise -> sends `0`

### `OsdMenuCtl(int en)`

Lightweight enable/disable:
- Enable: `spi_osd_cmd(0x28)` then `spi_osd_cmd(0x41)` — write-to-line-8 tells FPGA the line count
- Disable: `spi_osd_cmd(0x40)`

---

## 6. OSD Update / Flush

### `OsdUpdate()`

Transfers dirty lines to the FPGA:

```c
int n = is_menu() ? 19 : osd_size;
for (int i = 0; i < n; i++) {
    if (osdset & (1 << i)) {
        spi_osd_cmd_cont(OSD_CMD_WRITE | i);     // 0x20 | line
        spi_write(osdbuf + i * 256, 256, 0);     // 256 bytes, 8-bit mode
        DisableOsd();
        // Poll CD-ROM cores between lines to prevent audio dropouts
        if (is_megacd()) mcd_poll();
        if (is_pce()) pcecd_poll();
        if (is_saturn()) saturn_poll();
        if (is_neogeo_cd()) neocd_poll();
    }
}
osdset = 0;
```

Called from: `main.cpp` main loop, `scheduler.cpp` scheduler loop.

---

## 7. Complete SPI Transaction Formats

### Write OSD Line Data

```
EnableOsd()
  spi_b(0x20 | line_number)        // line 0-31 (5 bits)
  spi_b(byte[0])                    // 256 bytes of pixel column data
  spi_b(byte[1])
  ...
  spi_b(byte[255])
DisableOsd()
```

### Enable OSD (Menu Mode)

```
EnableOsd()
  spi_b(0x41 | flags)              // flags: 0x02=disable_kbd, 0x08=msg_window
DisableOsd()
```

### Enable OSD (Info Overlay)

```
EnableOsd()
  spi_b(0x45)                       // 0x41 | OSD_INFO(0x04)
  spi_w(x)                          // 16-bit X position on screen
  spi_w(y)                          // 16-bit Y position on screen
  spi_w(width)                      // 16-bit width in characters
  spi_w(height)                     // 16-bit height in characters
DisableOsd()
```

### Disable OSD

```
EnableOsd()
  spi_b(0x40)
DisableOsd()
```

### Set OSD Rotation

```
EnableOsd()
  spi_b(0x40)                       // OSD_CMD_DISABLE
  spi_w(0)                          // padding 1
  spi_w(0)                          // padding 2
  spi_w(0)                          // padding 3
  spi_w(0)                          // padding 4
  spi_w(rotation_value)             // 0, 1, or 3
DisableOsd()
```

FPGA distinguishes plain disable (just command byte) from rotation (command + 5 words) by the presence of additional data.

---

## 8. OSD Visual Elements

### Title Sidestripe

- Leftmost 22 bytes of every OSD line
- Contains core/menu title rendered **vertically** (rotated 90 degrees)
- White borders (0xFF) on left (3 bytes) and right (1 byte)
- 2-byte blue gap (0x00) separator
- Title text inverted (XOR'd with 0xFF): dark text on white background
- Vertically centered within available height
- Title reads in reverse order: line 0 gets bottom of title

### Navigation Arrows

On the last line (`n == osd_size - 1`):

**Left arrow** (when `OSD_ARROW_LEFT` set):
- At start of text area: 3 blank + char 0x10 (8) + char 0x14 (8) + 5 blank = 24 bytes
- Skips 3 characters from input string

**Right arrow** (when `OSD_ARROW_RIGHT` set):
- At end of line: 3 blank + char 0x15 (8) + char 0x11 (8) + 3 blank = 22 bytes
- Line limit reduced by 22 to make space

### Inversion (Highlighting)

- `invert != 0`: XOR all pixels with 0xFF
- `mininv`/`maxinv`: partial inversion range (character positions in 8-pixel units)

### Stipple (Dimming)

- When active: alternate columns masked with 0x55/0xAA
- Creates 50% dither/dimming effect
- Toggle mid-line with byte `0x0B` in string

### Inline Control Characters

| Byte | Effect |
|------|--------|
| `0x0B` | Toggle stipple on/off |
| `0x0C` | Toggle per-character XOR |
| `0x0D` | Carriage return -> next line |
| `0x0A` | Line feed -> next line |
| `0x00` | End of string |

### MiSTer Logo

- `logodata[10][227]`: 10 rows, 227 bytes each
- Rendered via `OsdDrawLogo(row)` with starfield overlay

---

## 9. Data Flow Summary

```
User Code (menu.cpp)
    |
    v
OsdSetTitle() -> titlebuffer[256] (rotated vertical title)
OsdSetSize()  -> osd_size (8 or 16)
OsdWrite()    -> osdbuf[] + osdset dirty bits
OsdDrawLogo() -> osdbuf[] with logo + stars
OSD_PrintInfo() -> osdbuf[] with info text
OsdClear()    -> zero osdbuf[], all dirty
    |
    v
OsdUpdate()   -> for each dirty line:
    |
    v
spi_osd_cmd_cont(0x20|line) -> spi_write(256 bytes, 8-bit) -> DisableOsd()
    |
    v
EnableOsd() -> fpga_spi_en(mask, 1) -> fpga_spi() bit-bang to hardware
    |
    v
FPGA OSD Controller -> renders overlay on video output
```

---

## 10. Constants Reference

```c
// Dimensions
#define OSDLINELEN    256           // Bytes per OSD line
#define INFO_MAXW     32            // Max info overlay width (chars)
#define INFO_MAXH     16            // Max info overlay height (lines)

// Timing
#define REPEATDELAY   500           // Key repeat delay (ms)
#define REPEATRATE    50            // Key repeat rate (ms)
#define SCROLL_DELAY  1000          // Scroll start delay (ms)
#define SCROLL_DELAY2 10            // Scroll step interval (ms)
#define BLANKSPACE    10            // Spaces between scroll repetitions

// Arrows
#define OSD_ARROW_LEFT   1
#define OSD_ARROW_RIGHT  2

// OSD targets
#define OSD_HDMI  1
#define OSD_VGA   2
#define OSD_ALL   3                 // (OSD_VGA | OSD_HDMI)
```

---

## 11. Global/Static State

| Variable | Type | Purpose |
|----------|------|---------|
| `osd_size` | `static int` | Number of OSD lines (8 or 16) |
| `osdbuf[256*32]` | `static uint8_t` | OSD framebuffer |
| `osdbufpos` | `static int` | Current write position |
| `osdset` | `static int` | Dirty-line bitmask (32 bits) |
| `framebuffer[16][256]` | `char` | Starfield overlay buffer |
| `stars[64]` | `struct star` | Star positions and velocities |
| `scroll_offset[2]` | `static unsigned long` | Scroll positions (2 independent) |
| `scroll_timer[2]` | `static unsigned long` | Scroll delay timers |
| `arrow` | `static int` | Current arrow flags |
| `titlebuffer[256]` | `static unsigned char` | Pre-rendered rotated title |
| `lastcorename[271]` | `static char` | Current core name string |
| `osd_target` | `static int` | OSD output target (spi.cpp) |
| `osd_is_visible` | `static char` | OSD visible flag (user_io.cpp) |

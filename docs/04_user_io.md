# User I/O Protocol (`user_io`) - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `user_io.h` | 292 | Constants, type definitions, public API |
| `user_io.cpp` | 4177 | Full implementation |

---

## 1. Complete SPI Command Table

All commands are sent via `SSPI_IO_EN` (EnableIO/DisableIO) unless noted otherwise.

### Primary I/O Commands (0x00 - 0x06)

| Command | Value | Direction | Payload | Description |
|---------|-------|-----------|---------|-------------|
| `UIO_STATUS` | `0x00` | HPS->FPGA | — | Status query |
| `UIO_BUT_SW` | `0x01` | HPS->FPGA | 16-bit map | Button state + config flags |
| `UIO_JOYSTICK0` | `0x02` | HPS->FPGA | 16 or 32 bits | Digital joystick 0 |
| `UIO_JOYSTICK1` | `0x03` | HPS->FPGA | 16 or 32 bits | Digital joystick 1 |
| `UIO_MOUSE` | `0x04` | HPS->FPGA | 3 x 16-bit words | Mouse data (PS/2 format) |
| `UIO_KEYBOARD` | `0x05` | HPS->FPGA | Variable | Keyboard scancode |
| `UIO_KBD_OSD` | `0x06` | HPS->FPGA | 8-bit code | OSD-only keycodes (Minimig) |

### Extended Joysticks (0x10 - 0x13)

| Command | Value | Direction | Payload |
|---------|-------|-----------|---------|
| `UIO_JOYSTICK2` | `0x10` | HPS->FPGA | 16/32 bits |
| `UIO_JOYSTICK3` | `0x11` | HPS->FPGA | 16/32 bits |
| `UIO_JOYSTICK4` | `0x12` | HPS->FPGA | 16/32 bits |
| `UIO_JOYSTICK5` | `0x13` | HPS->FPGA | 16/32 bits |

### Core Configuration (0x14 - 0x1F)

| Command | Value | Direction | Payload | Description |
|---------|-------|-----------|---------|-------------|
| `UIO_GET_STRING` | `0x14` | FPGA->HPS | Bytes until 0x00 | Read config string |
| `UIO_SET_STATUS` | `0x15` | HPS->FPGA | Legacy | Set status (legacy, superseded by 0x1E) |
| `UIO_GET_SDSTAT` | `0x16` | FPGA->HPS | See SD protocol | SD card emulation status |
| `UIO_SECTOR_RD` | `0x17` | HPS->FPGA | Block data | Sector data to core |
| `UIO_SECTOR_WR` | `0x18` | FPGA->HPS | Block data | Sector data from core |
| `UIO_SET_SDCONF` | `0x19` | HPS->FPGA | CSD(16)+CID(16)+1 | SD card configuration |
| `UIO_ASTICK` | `0x1a` | HPS->FPGA | joy#, X, Y | Left analog stick |
| `UIO_SIO_IN` | `0x1b` | HPS->FPGA | — | Serial data in |
| `UIO_SET_SDSTAT` | `0x1c` | HPS->FPGA | 8-bit | Mount/unmount notification |
| `UIO_SET_SDINFO` | `0x1d` | HPS->FPGA | 64-bit size | Mounted image info |
| `UIO_SET_STATUS2` | `0x1e` | HPS->FPGA | 8 x 16-bit words | 128-bit status |
| `UIO_GET_KBD_LED` | `0x1f` | FPGA->HPS | 16-bit | Keyboard LED control |

### Video/Audio/Display (0x20 - 0x41)

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `UIO_SET_VIDEO` | `0x20` | HPS->FPGA | Video settings |
| `UIO_PS2_CTL` | `0x21` | FPGA->HPS | PS/2 control bytes |
| `UIO_RTC` | `0x22` | HPS->FPGA | RTC data (MSM6242B BCD) |
| `UIO_GET_VRES` | `0x23` | FPGA->HPS | Video resolution |
| `UIO_TIMESTAMP` | `0x24` | HPS->FPGA | Unix epoch seconds |
| `UIO_LEDS` | `0x25` | HPS->FPGA | LEDs + emulation mode |
| `UIO_AUDVOL` | `0x26` | HPS->FPGA | Digital volume (bits to shift right) |
| `UIO_SETHEIGHT` | `0x27` | HPS->FPGA | Max vertical resolution |
| `UIO_GETUARTFLG` | `0x28` | FPGA->HPS | UART flags |
| `UIO_GET_STATUS` | `0x29` | FPGA->HPS | Status change detection |
| `UIO_SET_FLTCOEF` | `0x2A` | HPS->FPGA | Scaler filter coefficients |
| `UIO_SET_FLTNUM` | `0x2B` | HPS->FPGA | Scaler predefined filter |
| `UIO_GET_VMODE` | `0x2C` | FPGA->HPS | Video mode parameters |
| `UIO_SET_VPOS` | `0x2D` | HPS->FPGA | Video positions |
| `UIO_GET_OSDMASK` | `0x2E` | FPGA->HPS | OSD mask / SDRAM type |
| `UIO_SET_FBUF` | `0x2F` | HPS->FPGA | Frame buffer for HPS output |
| `UIO_WAIT_VSYNC` | `0x30` | HPS->FPGA | Wait for VSync |
| `UIO_SET_MEMSZ` | `0x31` | HPS->FPGA | Memory size to core |
| `UIO_SET_GAMMA` | `0x32` | HPS->FPGA | Gamma correction enable |
| `UIO_SET_GAMCURV` | `0x33` | HPS->FPGA | Gamma curve data |
| `UIO_CD_GET` | `0x34` | FPGA->HPS | CD status |
| `UIO_CD_SET` | `0x35` | HPS->FPGA | CD control |
| `UIO_INFO_GET` | `0x36` | FPGA->HPS | Info string index |
| `UIO_SETWIDTH` | `0x37` | HPS->FPGA | Max horizontal resolution |
| `UIO_SETSYNC` | `0x38` | HPS->FPGA | Sync settings |
| `UIO_SET_AFILTER` | `0x39` | HPS->FPGA | Audio filter |
| `UIO_SET_AR_CUST` | `0x3A` | HPS->FPGA | Custom aspect ratio |
| `UIO_SET_UART` | `0x3B` | HPS->FPGA | UART mode + baud |
| `UIO_CHK_UPLOAD` | `0x3C` | HPS->FPGA | Check upload |
| `UIO_ASTICK_2` | `0x3D` | HPS->FPGA | Right analog stick |
| `UIO_SHADOWMASK` | `0x3E` | HPS->FPGA | Shadow mask data |
| `UIO_GET_RUMBLE` | `0x3F` | FPGA->HPS | Rumble feedback |
| `UIO_GET_FB_PAR` | `0x40` | FPGA->HPS | Frame buffer parameters |
| `UIO_SET_YC_PAR` | `0x41` | HPS->FPGA | Y/C parameters |

### File I/O Commands (via `SSPI_FPGA_EN`, NOT `SSPI_IO_EN`)

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `FIO_FILE_TX` | `0x53` | HPS->FPGA | File transfer control (0xFF=download, 0xAA=upload, 0x00=stop) |
| `FIO_FILE_TX_DAT` | `0x54` | Bidirectional | File transfer data payload |
| `FIO_FILE_INDEX` | `0x55` | HPS->FPGA | Set file index |
| `FIO_FILE_INFO` | `0x56` | HPS->FPGA | File extension (4 uppercase chars) |

### DMA Commands

| Command | Value | Direction |
|---------|-------|-----------|
| `UIO_DMA_WRITE` | `0x61` | HPS->FPGA |
| `UIO_DMA_READ` | `0x62` | FPGA->HPS |
| `UIO_DMA_SDIO` | `0x63` | Bidirectional |

### Minimig v2 Commands (0xF0 - 0xF9)

| Command | Value | Purpose |
|---------|-------|---------|
| `UIO_MM2_WR` | `0xF0` | Minimig write |
| `UIO_MM2_RST` | `0xF1` | Minimig reset |
| `UIO_MM2_AUD` | `0xF2` | Minimig audio config |
| `UIO_MM2_CHIP` | `0xF3` | Minimig chipset config |
| `UIO_MM2_CPU` | `0xF4` | Minimig CPU config |
| `UIO_MM2_MEM` | `0xF5` | Minimig memory config |
| `UIO_MM2_VID` | `0xF6` | Minimig video config |
| `UIO_MM2_FLP` | `0xF7` | Minimig floppy config |
| `UIO_MM2_HDD` | `0xF8` | Minimig HDD config |
| `UIO_MM2_JOY` | `0xF9` | Minimig joystick config |

---

## 2. Detailed Protocol Formats

### 2.1 Button/Config Map (UIO_BUT_SW)

Sent via `spi_uio_cmd16(UIO_BUT_SW, map)`:

```
Bit  Constant                Value    Meaning
---  --------                -----    -------
0    BUTTON1                 0x0001   OSD/Menu button currently pressed
1    BUTTON2                 0x0002   User button pressed (or keyboard reset active)
2    CONF_VGA_SCALER         0x0004   VGA scaler enabled
3    CONF_CSYNC              0x0008   Composite sync enabled
4    CONF_FORCED_SCANDOUBLER 0x0010   Forced scandoubler
5    CONF_YPBPR              0x0020   YPbPr mode
6    CONF_AUDIO_96K          0x0040   96kHz audio mode
7    CONF_DVI                0x0080   DVI mode (no audio over HDMI)
8    CONF_HDMI_LIMITED1      0x0100   HDMI limited range bit 0
9    CONF_VGA_SOG            0x0200   VGA Sync on Green
10   CONF_DIRECT_VIDEO       0x0400   Direct video output mode
11   CONF_HDMI_LIMITED2      0x0800   HDMI limited range bit 1
12   CONF_VGA_FB             0x1000   VGA framebuffer mode
13   CONF_DIRECT_VIDEO2      0x2000   Direct video mode 2
```

### 2.2 Joystick Data

#### Digital Joystick Bit Map

```
Bit  Constant    Value    Meaning
---  --------    -----    -------
0    JOY_RIGHT   0x0001   D-pad Right
1    JOY_LEFT    0x0002   D-pad Left
2    JOY_DOWN    0x0004   D-pad Down
3    JOY_UP      0x0008   D-pad Up
4    JOY_A       0x0010   A / Button 1
5    JOY_B       0x0020   B / Button 2
6    JOY_SELECT  0x0040   Select / Button 3
7    JOY_START   0x0080   Start / Button 4
8    JOY_X       0x0100   X
9    JOY_Y       0x0200   Y
10   JOY_L       0x0400   Left bumper
11   JOY_R       0x0800   Right bumper
12   JOY_L2      0x1000   Left trigger
13   JOY_R2      0x2000   Right trigger
14   JOY_L3      0x4000   Left stick click
15   JOY_R3      0x8000   Right stick click
```

#### Protocol

```
EnableIO()
  spi_w(UIO_JOYSTICK0 + joy)    // joy 0-1, or UIO_JOYSTICK2 + (joy-2) for 2-5
  spi_w(bitmask & 0xFFFF)       // Low 16 bits
  [spi_w(bitmask >> 16)]        // High 16 bits (only if any high bits ever used)
DisableIO()
```

**Joystick swap:** Joysticks 0 and 1 are swapped via XOR with 1 when `joyswap` flag is set.

**Digital-to-analog translation:** When `joy_transl == 1` and direction changes, also calls `user_io_l_analog_joystick()` with: left=-128, right=+127, up=-128, down=+127, neutral=0.

#### Analog Stick Protocol

```
EnableIO()
  spi_b(UIO_ASTICK)             // 0x1A for left stick, 0x3D for right stick
  spi_b(joystick_number)
  spi_w((valueY << 8) | (uint8_t)valueX)   // if io_ver >= 1 (16-bit word mode)
  // OR: spi8(valueX); spi8(valueY);        // if io_ver == 0 (byte mode)
DisableIO()
```

Values: -128 (full left/up) to +127 (full right/down), 0 = center.

### 2.3 Keyboard Protocol

#### Generic 8BIT Core (PS/2 Scancodes)

```
EnableIO()
  spi_w(UIO_KEYBOARD)           // 0x05
  [spi8(0xE0)]                  // Extended key prefix (if key is extended)
  [spi8(0xF0)]                  // Break prefix for key release (scan set 2)
  spi8(scancode)                // PS/2 scancode byte
DisableIO()
```

**Scan set 1 alternative:** Break = `scancode | 0x80` (no 0xF0 prefix).

**Special key sequences (scan set 2):**
- Pause make: `E1,14,77,E1,F0,14,F0,77`
- Pause break: (none)
- Print Screen make: `E0,12,E0,7C`
- Print Screen break: `E0,F0,7C,E0,F0,12`

**Scan set 1 special sequences:**
- Pause make: `E1,1D,45,E1,9D,C5`
- Print Screen make: `E0,2A,E0,37`
- Print Screen break: `E0,B7,E0,AA`

#### Minimig (Amiga Keycodes)

```
EnableIO()
  spi_b(UIO_KEYBOARD)
  spi_b(amiga_keycode)          // bit 7 = 1 for break (release)
DisableIO()
```

Uses `get_amiga_code()` to translate USB HID keycodes to Amiga scancodes.
Caps Lock handled as toggle: alternating make/break on each press.
Rate-limited with 16-deep FIFO and 10ms minimum between keys.

#### Keyboard Reply (PS/2 Responses)

```c
spi_uio_cmd16(UIO_KEYBOARD, 0xFF00 | reply_code);
```

| Reply | Value | Meaning |
|-------|-------|---------|
| ACK | `0xFA` | Command acknowledged |
| BAT | `0xAA` | Basic Assurance Test passed |
| RESEND | `0xFE` | Request to resend |
| ECHO | `0xEE` | Echo response |
| Device ID | `0xAB, 0x83` | PS/2 keyboard ID (sent as two replies) |

### 2.4 Mouse Protocol

#### Generic 8BIT Core (PS/2 Mouse Format)

PS/2 byte 0 construction:
```
bit 0 = Left button
bit 1 = Right button
bit 2 = Middle button
bit 3 = Always 1
bit 4 = X sign bit
bit 5 = Y sign bit (Y is inverted)
bit 6 = X overflow
bit 7 = Y overflow
```

Protocol:
```
EnableIO()
  spi_w(UIO_MOUSE)                                    // 0x04
  spi_w(ps2_byte0 | ((wheel & 0x7F) << 8))           // byte0 + wheel in high byte
  spi_w(deltaX | (((buttons) << 5) & 0xF00))         // X + extra buttons [7:5]
  spi_w((-deltaY) | (((buttons) << 1) & 0x100))      // -Y + extra button [4]
DisableIO()
```

#### Minimig Mouse

```
EnableIO()
  spi_b(UIO_MOUSE)
  spi_b(clamp(deltaX, -127, 127))     // 8-bit signed X
  spi_b(clamp(deltaY, -127, 127))     // 8-bit signed Y
  spi_b(buttons & 0x07)                // 3 buttons
  spi_b(clamp(wheel, -127, 127))       // 8-bit signed wheel
DisableIO()
```

#### Mouse Reply (PS/2 Responses)

```c
spi_uio_cmd16(UIO_MOUSE, 0xFF00 | reply_code);
```

### 2.5 Status Protocol (128-bit)

The core status is a 128-bit (16-byte) value stored in `cur_status[16]`.

#### Send Status to FPGA

```
EnableIO()
  spi_w(UIO_SET_STATUS2)               // 0x1E
  for (i = 0; i < 16; i += 2)
    spi_w((cur_status[i+1] << 8) | cur_status[i])   // 8 x 16-bit words, LE
DisableIO()
```

#### Read Status Change from FPGA

```
stchg = spi_uio_cmd_cont(UIO_GET_STATUS)    // 0x29
if ((stchg & 0xF0) == 0xA0) {
    if ((stchg & 0xF) != last_counter) {
        // Core updated its status — read 128 bits:
        for (i = 0; i < 16; i += 2) {
            x = spi_w(0);
            cur_status[i] = (char)x;
            cur_status[i+1] = (char)(x >> 8);
        }
    }
}
DisableIO()
```

The FPGA returns `0xA0 | counter` where `counter` (4-bit) increments each time the core changes its status.

#### Status Bit Addressing

Options in the config string use bracket notation: `[high:low]` or `[bit]`.
- `"[3:1]"` = bits 3 down to 1 (3 bits)
- `"[0]"` = bit 0 (1 bit)
- Extended status (`ex=1`) adds 32 to positions

Maximum 8 bits per option field.

### 2.6 Config String Protocol

```
EnableIO()
  spi_w(UIO_GET_STRING)      // 0x14
  loop:
    byte = spi_in()          // Read one byte (sends 0 to clock it out)
    if (byte == 0) break
    cfgstr[pos++] = byte
  // Store in cfgstr[10240]
DisableIO()
```

The config string is semicolon-delimited. Each segment is accessed by index:
- Index 0: Core name
- Index 1: Special parameters
- Index 2+: Menu entry definitions

### 2.7 SD Card Emulation Protocol

#### Status Poll — New Format (bit 15 set)

```
c = spi_uio_cmd_cont(UIO_GET_SDSTAT)   // 0x16
if (c & 0x8000) {
    disk  = (c >> 2) & 0xF             // Disk index 0-15
    op    = c & 3                       // 1 = read request, 2 = write request
    blks  = ((c >> 9) & 0x3F) + 1      // Block count 1-64
    blksz = 128 << ((c >> 6) & 7)      // Block size: 128, 256, 512, 1024, ... 16384
    // Special: PSX disk 1 uses blksz = 2352
    spi_w(0)                            // Padding word
    lba = spi_w(0) | (spi_w(0) << 16)  // 32-bit LBA
    ack = disk << 8                     // Acknowledge bits
}
DisableIO()
```

#### Status Poll — Legacy Format (bit 15 clear)

```
c = spi_w(0)                            // Read second word
if ((c & 0xF0) == 0x50) {
    lba = spi_w(0) | (spi_w(0) << 16)  // 32-bit LBA
    // Decode disk + operation from bit fields:
    // c & 0x0003: disk 0 read/write
    // c & 0x0900: disk 1 read/write
    // c & 0x1200: disk 2 read/write
    // c & 0x2400: disk 3 read/write
    // c & 0x000C == 0x0C: SD config request
    blksz = 512; blks = 1              // Always single 512-byte blocks
}
DisableIO()
```

#### Sector Read (HPS sends data to FPGA)

```
EnableIO()
  spi_w(UIO_SECTOR_RD | ack)           // 0x17 | (disk << 8)
  spi_block_write(buffer, fio_size, block_size)
DisableIO()
```

#### Sector Write (FPGA sends data to HPS)

```
EnableIO()
  spi_w(UIO_SECTOR_WR | ack)           // 0x18 | (disk << 8)
  spi_block_read(buffer, fio_size, block_size)
DisableIO()
```

#### SD Card Mount Notification

```
EnableIO()
  spi_b(UIO_SET_SDSTAT)                // 0x1C
  spi_b((1 << disk_index) | (readonly ? 0x80 : 0))
DisableIO()
```

#### SD Card Image Size

```
EnableIO()
  spi_b(UIO_SET_SDINFO)                // 0x1D
  if (io_ver) {
      spi32_w(size & 0xFFFFFFFF)       // Low 32 bits (word mode)
      spi32_w(size >> 32)              // High 32 bits
  } else {
      spi32_b(size & 0xFFFFFFFF)       // Low 32 bits (byte mode)
      spi32_b(size >> 32)              // High 32 bits
  }
DisableIO()
```

#### SD Card CSD/CID Configuration

```
EnableIO()
  spi_w(UIO_SET_SDCONF)                // 0x19
  spi_write(CID, 16, fio_size)         // 16 bytes Card ID
  spi_write(CSD, 16, fio_size)         // 16 bytes Card-Specific Data
  spi_b(1)                             // SDHC flag (always 1)
DisableIO()
```

### 2.8 File Transfer Protocol

#### Download (HPS to FPGA)

```
// 1. Set file index
EnableFpga()
  spi8(FIO_FILE_INDEX)                  // 0x55
  spi8(index)                           // 8-bit index (or spi_w for 16-bit)
DisableFpga()

// 2. Send file extension info
EnableFpga()
  spi8(FIO_FILE_INFO)                   // 0x56
  spi_w((char1 << 8) | char2)          // First 2 chars (uppercase)
  spi_w((char3 << 8) | char4)          // Last 2 chars
DisableFpga()

// 3. Signal download start
EnableFpga()
  spi8(FIO_FILE_TX)                     // 0x53
  spi8(0xFF)                            // 0xFF = download mode
  [spi_w(load_addr & 0xFFFF)]          // Optional: load address low 16
  [spi_w(load_addr >> 16)]             // Optional: load address high 16
DisableFpga()

// 4. Send data (repeated for each chunk)
EnableFpga()
  spi8(FIO_FILE_TX_DAT)                // 0x54
  spi_write(data, length, fio_size)    // fio_size: 0=8-bit, 1=16-bit
DisableFpga()

// 5. Signal download complete
EnableFpga()
  spi8(FIO_FILE_TX)                     // 0x53
  spi8(0x00)                            // 0x00 = transfer complete
DisableFpga()
```

#### Upload (FPGA to HPS)

Same as download but:
- Step 3: sends `0xAA` instead of `0xFF`
- Step 4: uses `spi_read()` instead of `spi_write()`

### 2.9 RTC Protocol

#### MSM6242B BCD Format (type & 1)

```
EnableIO()
  spi_w(UIO_RTC)                        // 0x22
  spi_w((minutes_bcd << 8) | seconds_bcd)
  spi_w((day_bcd << 8) | hours_bcd)
  spi_w((year_bcd << 8) | month_bcd)
  spi_w((0x40 << 8) | weekday)          // 0x40 flag + day of week
DisableIO()
```

#### Unix Timestamp (type & 2)

```
EnableIO()
  spi_w(UIO_TIMESTAMP)                  // 0x24
  spi_w(epoch_seconds & 0xFFFF)         // Low 16 bits (local time)
  spi_w(epoch_seconds >> 16)            // High 16 bits
DisableIO()
```

### 2.10 UART Protocol

```
EnableIO()
  spi_w(UIO_SET_UART)                   // 0x3B
  spi_w(mode)                           // UART mode (1-5, with 4/5 mapped to 1)
  spi_w(baud & 0xFFFF)                  // Baud rate low 16
  spi_w(baud >> 16)                     // Baud rate high 16
DisableIO()
```

### 2.11 PS/2 Control Protocol (UIO_PS2_CTL)

```
EnableIO()
  spi_w(UIO_PS2_CTL)                    // 0x21
  c1 = spi_w(0)                         // Keyboard: low 8 = cmd, bit 8 = data valid
  c2 = spi_w(0)                         // Mouse: low 8 = cmd, bit 8>>1 = data valid
DisableIO()
// Returns: bit 0 = kbd cmd pending, bit 1 = mouse cmd pending
```

#### PS/2 Keyboard Commands Handled

| Command | Response | Action |
|---------|----------|--------|
| `0xFF` (Reset) | FA, AA | Reset scan set to 2 |
| `0xF6` (Set Default) | FA | Reset scan set to 2 |
| `0xF5`/`0xF4` (Disable/Enable) | FA | — |
| `0xF3` (Set Typematic) | FA | Read next byte for rate |
| `0xF2` (Get ID) | FA, AB, 83 | PS/2 keyboard ID |
| `0xF0` (Set Scan Set) | FA | Next byte: 0=get, 1-3=set |
| `0xEE` (Echo) | EE | — |
| `0xED` (Set LEDs) | FA | Next byte: bit2=Caps |
| Other | FE (Resend) | — |

#### PS/2 Mouse Commands Handled

| Command | Response | Action |
|---------|----------|--------|
| `0xFF` (Reset) | FA, AA, 00 | — |
| `0xF6`/`0xF5`/`0xF4`/`0xF0`/`0xEA`/`0xE6` | FA | — |
| `0xF3` (Sample Rate) | FA | Read next byte |
| `0xF2` (Get ID) | FA, 00 | Mouse ID |
| `0xE9` (Status) | FA, 00, 00, 00 | — |
| `0xE8` (Set Resolution) | FA | Read next byte |
| Other | FE (Resend) | — |

### 2.12 LED/Emulation Mode

```
spi_uio_cmd16(UIO_LEDS, 0x6000 | emu_led);
```

| emu_led | Mode |
|---------|------|
| `0x00` | EMU_NONE (normal keyboard) |
| `0x20` | EMU_JOY0 (keyboard -> joystick 0) |
| `0x40` | EMU_JOY1 (keyboard -> joystick 1) |
| `0x60` | EMU_MOUSE (keyboard -> mouse) |

---

## 3. Keyboard LED Control

Read via `UIO_GET_KBD_LED`:

```
Bit  Constant              Value  Meaning
---  --------              -----  -------
0    KBD_LED_CAPS_CONTROL  0x01   Core wants to control Caps Lock LED
1    KBD_LED_CAPS_STATUS   0x02   Core's desired Caps Lock state
2    KBD_LED_NUM_CONTROL   0x04   Core wants to control Num Lock LED
3    KBD_LED_NUM_STATUS    0x08   Core's desired Num Lock state
4    KBD_LED_SCRL_CONTROL  0x10   Core wants to control Scroll Lock LED
5    KBD_LED_SCRL_STATUS   0x20   Core's desired Scroll Lock state
6    KBD_LED_FLAG_STATUS   0x40   Valid LED status data flag
```

If bit 8 of the return word is set, it indicates PS/2 control mode (`use_ps2ctl = 1`).

---

## 4. Core Initialization (`user_io_init`)

Called from `main()` with the core path and optional XML path.

1. Set `core_path` and `rbf_path`
2. Read `fpga_core_id()` -> core type byte
3. Read `fpga_get_fio_size()` -> 0=8-bit, 1=16-bit file I/O
4. Read `fpga_get_io_version()` -> I/O version (0-3)
5. Handle `CORE_TYPE_8BIT2`: remap to `CORE_TYPE_8BIT`, set `dual_sdr=1`
6. If arcade XML: set `is_arcade_type=1`, call `arcade_pre_parse()`
7. For 8BIT cores: send SDRAM size via `spi_uio_cmd32(UIO_SET_MEMSZ, ...)`, assert reset
8. Read config string from FPGA via `UIO_GET_STRING`
9. Read core name (confstr index 0)
10. Parse INI configuration (`cfg_parse()`)
11. Initialize IDE if needed
12. Parse config string options:
    - `J` -> joystick button names and count
    - `SS` -> save state base address and size
    - `U` -> UART speed options
    - `C` -> cheats enabled
    - `V` -> config version
    - `X` -> OSD disabled
13. Core-specific initialization:
    - Minimig -> `BootInit()`
    - x86 -> `x86_init()`
    - Archie -> `archie_init()`
    - SharpMZ -> `sharpmz_init()`
    - Arcade -> `arcade_send_rom()`
    - Generic -> load boot ROMs (confstr `P` option) and mount VHDs
14. Load saved status from `.CFG` file
15. Release core reset
16. Setup UART mode

---

## 5. Main Poll Loop (`user_io_poll`)

Called repeatedly from the scheduler. For 8BIT/SHARPMZ core types:

1. **Buttons:** `user_io_send_buttons(0)` — push button/config state to FPGA
2. **Status changes:** `check_status_change()` via `UIO_GET_STATUS` — detect core-initiated status updates
3. **SD card emulation:** Poll `UIO_GET_SDSTAT`, handle sector read/write for up to 16 disks with 16KB per-disk read-ahead buffers
4. **Keyboard LEDs:** Every 100ms, poll `UIO_GET_KBD_LED` or handle PS/2 control commands
5. **Core info:** Poll `UIO_INFO_GET` for info overlay text
6. **Video mode:** Every 500ms-1s, poll `UIO_GET_OSDMASK` for SDRAM config (menu core)
7. **Per-core polling:** MegaCD, PCE CD, Saturn, PSX, NeoGeo CD, N64, Archie, SharpMZ, x86
8. **Save states:** Process save state changes (poll shared memory counters)
9. **Disk LED:** Turn off LED after 50ms timeout

---

## 6. Core Type Detection

All lazy-initialized, cached comparison of `orig_name` against known core names:

```c
is_minimig()     // "minimig"       is_menu()        // "MENU"
is_x86()         // "AO486"         is_snes()        // "SNES"
is_sgb()         // "SGB"           is_neogeo()      // "neogeo"
is_neogeo_cd()   // is_neogeo() && neocd_is_en()
is_megacd()      // "MEGACD"        is_pce()         // "TGFX16"
is_archie()      // "ARCHIE"        is_gba()         // "GBA"
is_c64()         // "C64"           is_psx()         // "PSX"
is_st()          // "AtariST"       is_pcxt()        // "PCXT"
is_saturn()      // "Saturn"        is_n64()         // "N64"
is_uneon()       // "Uneon"         is_arcade()      // set by XML/MRA detection
is_sharpmz()     // core_type == CORE_TYPE_SHARPMZ
has_menu()       // core_name is non-empty (and OSD not disabled)
```

---

## 7. Key Public API Functions

### File Transfer

```c
void user_io_set_index(unsigned char index);       // Send 8-bit file index
void user_io_set_aindex(uint16_t index);           // Send 16-bit file index
void user_io_set_download(unsigned char enable, int addr = 0);  // Start/stop download
void user_io_file_tx_data(const uint8_t *addr, uint32_t len);  // Send data chunk
void user_io_set_upload(unsigned char enable, int addr = 0);    // Start/stop upload
void user_io_file_rx_data(uint8_t *addr, uint32_t len);        // Receive data chunk
void user_io_file_info(const char *ext);           // Send file extension
int  user_io_file_tx(const char *name, ...);       // High-level file transmit
int  user_io_file_tx_a(const char *name, uint16_t index);  // 16-bit index variant
```

### SD Card / Disk

```c
int  user_io_file_mount(const char *name, unsigned char index, char pre, int pre_size);
void user_io_bufferinvalidate(unsigned char index);
```

### Status

```c
uint32_t user_io_status_get(const char *opt, int ex = 0);
void     user_io_status_set(const char *opt, uint32_t value, int ex = 0);
int      user_io_status_save(const char *filename);
void     user_io_status_reset();
```

### Input

```c
void user_io_digital_joystick(unsigned char joystick, uint64_t map, int newdir);
void user_io_l_analog_joystick(unsigned char joystick, char valueX, char valueY);
void user_io_r_analog_joystick(unsigned char joystick, char valueX, char valueY);
void user_io_mouse(unsigned char b, int16_t x, int16_t y, int16_t w);
void user_io_kbd(uint16_t key, int press);
```

### Configuration

```c
void     user_io_read_confstr();
char    *user_io_get_confstr(int index);
unsigned char user_io_core_type();
char    *user_io_get_core_name(int orig = 0);
char    *user_io_get_core_path(const char *suffix = NULL, int recheck = 0);
```

### Polling & Control

```c
void user_io_poll();                    // Main poll loop (called from scheduler)
void user_io_send_buttons(char force);  // Push button/config state
void user_io_rtc_reset();              // Force RTC update
void user_io_osd_key_enable(char on);  // Route keys to OSD vs core
```

---

## 8. Static State Variables

```c
static char core_path[1024]            // Path to core RBF or MRA
static char rbf_path[1024]            // Path to RBF file
static fileTYPE sd_image[16]          // Up to 16 mounted disk images
static uint64_t buffer_lba[16]        // Per-disk read-ahead LBA cache
static unsigned char core_type        // Current core type byte
static unsigned char dual_sdr         // Dual SDRAM flag
static int fio_size                   // File I/O width (0=8-bit, 1=16-bit)
static int io_ver                     // I/O version from FPGA
static char cur_status[16]            // 128-bit status (16 bytes)
static char cfgstr[10240]             // Cached config string
static int emu_mode                   // Keyboard emulation mode (0-3)
static int joyswap                    // Joystick swap flag
static int joy_transl                 // Joystick translation mode
static char osd_is_visible            // OSD visible flag
static uint16_t sdram_cfg             // SDRAM module type
static uint32_t uart_mode             // UART mode flags
static char core_name[32]             // Effective core name
static char orig_name[32]             // Original core name from config string
static char ovr_name[32]             // Override name (from MRA)
static uint8_t use_ps2ctl            // PS/2 control mode active
static unsigned long rtc_timer       // RTC update timer
static uint32_t file_crc             // CRC32 of last transmitted file
```

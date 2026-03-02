# SPI Communication Protocol - Complete Reference

## Overview

The "SPI" in MiSTer is **NOT hardware SPI**. It is a **software-emulated SPI protocol** implemented via the FPGA Manager's GPO and GPI registers. The HPS (ARM) writes 16-bit words to GPO[15:0] and toggles a strobe bit (GPO bit 17). The FPGA fabric reads the data and responds via GPI[15:0].

## Source Files

| File | Lines | Role |
|------|-------|------|
| `fpga_io.h` | 44 | Low-level SPI primitives API |
| `fpga_io.cpp` | 1010 | SPI bit-bang implementation |
| `spi.h` | 63 | Higher-level SPI protocol API |
| `spi.cpp` | 205 | Protocol layer implementation |

---

## 1. Physical Layer

### Registers

| Register | Address | Direction | Purpose |
|----------|---------|-----------|---------|
| GPO | `0xFF706010` | HPS->FPGA | 32-bit: data[15:0] + control bits |
| GPI | `0xFF706014` | FPGA->HPS | 32-bit: data[15:0] + status bits |

### GPO Bit Assignments for SPI

```
Bit(s)   Constant        Purpose
------   --------        -------
[15:0]   (data)          16-bit word being sent to FPGA
[17]     SSPI_STROBE     SPI clock strobe — toggled to clock data
[18]     SSPI_FPGA_EN    Chip-select: FPGA core (file I/O channel)
[19]     SSPI_OSD_EN     Chip-select: OSD subsystem
[20]     SSPI_IO_EN      Chip-select: User I/O subsystem
[31]     (active mode)   Must be set to 1 during SPI transactions
```

### GPI Bit Assignments for SPI

```
Bit(s)   Constant        Purpose
------   --------        -------
[15:0]   (data)          16-bit response word from FPGA
[17]     SSPI_ACK        SPI acknowledge — FPGA sets high after strobe, clears after strobe removed
[31]     (ready flag)    0 = FPGA in user mode (ready), 1 = uninitialized
```

---

## 2. Two SPI Transfer Modes

### Handshake Mode (`fpga_spi()`) — Reliable

This mode uses a full request/acknowledge handshake:

```
Step  GPO Action                  GPI Check                    Description
----  ----------                  ---------                    -----------
1     Write data[15:0], STROBE=0  —                           Place data word, clear strobe
2     Set STROBE=1 (bit 17)       —                           Assert clock strobe
3     —                           Poll until ACK=1 (bit 17)   Wait for FPGA to latch data
4     Clear STROBE=0              —                           De-assert strobe
5     —                           Poll until ACK=0 (bit 17)   Wait for FPGA to complete
6     —                           Read GPI[15:0]              Capture response word
```

**Safety:** If GPI bit 31 is set during polling (FPGA uninitialized), calls `fpga_wait_to_reset()` and returns 0.

**Use case:** All normal command/response transactions where reliability is required.

### Fast Mode (`fpga_spi_fast()`) — High Throughput

This mode skips the acknowledge handshake:

```
Step  GPO Action                  GPI Check           Description
----  ----------                  ---------           -----------
1     Write data[15:0], STROBE=0  —                  Place data word, clear strobe
2     Set STROBE=1 (bit 17)       —                  Assert clock strobe
3     Clear STROBE=0              —                  Immediately de-assert (no polling)
4     —                           Read GPI[15:0]     Capture response word
```

**Relies on:** The FPGA being fast enough to respond within the write-read latency of the memory-mapped bus. Much faster but less safe.

**Use case:** Block data transfers where throughput matters and the FPGA is known to keep up.

---

## 3. Chip Select Lines

Three chip-select bits in GPO multiplex access to different FPGA subsystems:

| Subsystem | GPO Bit | Constant | Enable/Disable Functions |
|-----------|---------|----------|--------------------------|
| FPGA Core | 18 | `SSPI_FPGA_EN` | `EnableFpga()` / `DisableFpga()` |
| OSD | 19 (+18,20) | `SSPI_OSD_EN` | `EnableOsd()` / `DisableOsd()` |
| User I/O | 20 | `SSPI_IO_EN` | `EnableIO()` / `DisableIO()` |

### Chip Select Protocol

Before starting any SPI transaction, the appropriate chip-select must be asserted. After the transaction, it must be de-asserted.

```
EnableFpga()         // Assert SSPI_FPGA_EN
  fpga_spi(cmd)      // Send command
  fpga_spi(data)     // Send/receive data
  ...
DisableFpga()        // De-assert SSPI_FPGA_EN
```

### OSD Chip Select Logic (Special)

OSD can target HDMI, VGA, or both outputs. The chip-select mask is constructed accordingly:

```c
void EnableOsd() {
    if (!(osd_target & OSD_ALL)) osd_target = OSD_ALL;
    uint32_t mask = SSPI_OSD_EN | SSPI_IO_EN | SSPI_FPGA_EN;  // start with all 3
    if (osd_target & OSD_HDMI) mask &= ~SSPI_FPGA_EN;  // HDMI: don't select FPGA
    if (osd_target & OSD_VGA)  mask &= ~SSPI_IO_EN;    // VGA: don't select IO
    fpga_spi_en(mask, 1);
}
```

| Target | `OSD_EN` | `IO_EN` | `FPGA_EN` | Effect |
|--------|----------|---------|-----------|--------|
| `OSD_ALL` (3) | ON | OFF | OFF | Both HDMI + VGA OSD |
| `OSD_HDMI` (1) | ON | ON | OFF | HDMI OSD only |
| `OSD_VGA` (2) | ON | OFF | ON | VGA OSD only |

```c
void DisableOsd() {
    fpga_spi_en(SSPI_OSD_EN | SSPI_IO_EN | SSPI_FPGA_EN, 0);  // disable all 3
}
```

### OSD Target Setting

```c
#define OSD_HDMI  1
#define OSD_VGA   2
#define OSD_ALL   (OSD_VGA | OSD_HDMI)   // = 3

void EnableOsd_on(int target);   // Sets osd_target for subsequent EnableOsd() calls
```

---

## 4. SPI Primitive Functions (`spi.h`)

### Inline Helpers

```c
// Send/receive single byte (places byte in lower 8 bits of 16-bit SPI word)
static inline uint8_t spi_b(uint8_t parm) {
    return (uint8_t)fpga_spi(parm);
}

// Send/receive single 16-bit word
static inline uint16_t spi_w(uint16_t word) {
    return fpga_spi(word);
}

// Receive byte (sends 0)
static inline uint8_t spi_in() {
    return (uint8_t)fpga_spi(0);
}

// Alias
#define spi8(x) spi_b(x)
```

### 32-bit SPI Functions (`spi.cpp`)

```c
// Send/receive 32-bit word as two 16-bit SPI words (little-endian)
uint32_t spi32_w(uint32_t parm) {
    uint32_t res = spi_w((uint16_t)parm);
    res |= spi_w(parm >> 16) << 16;
    return res;
}

// Send 32-bit word as four 8-bit SPI bytes (LSB first)
void spi32_b(uint32_t parm) {
    spi_b((uint8_t)parm);
    spi_b((uint8_t)(parm >> 8));
    spi_b((uint8_t)(parm >> 16));
    spi_b((uint8_t)(parm >> 24));
}
```

### Block Transfer Functions (`spi.cpp`)

```c
// Read block of bytes or words
void spi_read(uint8_t *addr, uint32_t len, int wide)
    // wide=1: reads as 16-bit words via spi_w(0)
    // wide=0: reads as 8-bit bytes via spi_b(0)

// Write block of bytes or words
void spi_write(const uint8_t *addr, uint32_t len, int wide)
    // wide=1: writes as 16-bit words via spi_w()
    // wide=0: writes as 8-bit bytes via spi_b()

// Fast block read (uses fpga_spi_fast_block_* for maximum throughput)
void spi_block_read(uint8_t *addr, int wide, int sz = 512)
    // wide=1: calls fpga_spi_fast_block_read() (16-bit mode)
    // wide=0: calls fpga_spi_fast_block_read_8() (8-bit mode)

// Fast block write
void spi_block_write(const uint8_t *addr, int wide, int sz = 512)
    // wide=1: calls fpga_spi_fast_block_write() (16-bit mode)
    // wide=0: calls fpga_spi_fast_block_write_8() (8-bit mode)
```

---

## 5. OSD SPI Command Functions (`spi.cpp`)

These handle the OSD chip-select and send OSD-specific commands:

```c
// Send OSD command byte, leave chip-select asserted (for follow-up data)
void spi_osd_cmd_cont(uint8_t cmd) {
    EnableOsd();
    spi_b(cmd);
}

// Send OSD command byte, then de-assert chip-select
void spi_osd_cmd(uint8_t cmd) {
    spi_osd_cmd_cont(cmd);
    DisableOsd();
}

// Send OSD command + 1 parameter byte, leave asserted
void spi_osd_cmd8_cont(uint8_t cmd, uint8_t parm) {
    EnableOsd();
    spi_b(cmd);
    spi_b(parm);
}

// Send OSD command + 1 parameter byte, then de-assert
void spi_osd_cmd8(uint8_t cmd, uint8_t parm) {
    spi_osd_cmd8_cont(cmd, parm);
    DisableOsd();
}
```

---

## 6. User I/O SPI Command Functions (`spi.cpp`)

These handle the IO chip-select and send user I/O commands:

```c
// Send 16-bit command, leave chip-select asserted, return first response word
uint16_t spi_uio_cmd_cont(uint16_t cmd) {
    EnableIO();
    return spi_w(cmd);
}

// Send 16-bit command, de-assert chip-select, return response
uint16_t spi_uio_cmd(uint16_t cmd) {
    uint16_t res = spi_uio_cmd_cont(cmd);
    DisableIO();
    return res;
}

// Send 8-bit command + 8-bit parameter, leave asserted
uint8_t spi_uio_cmd8_cont(uint8_t cmd, uint8_t parm) {
    EnableIO();
    spi_b(cmd);
    return spi_b(parm);
}

// Send 8-bit command + 8-bit parameter, de-assert
uint8_t spi_uio_cmd8(uint8_t cmd, uint8_t parm) {
    uint8_t res = spi_uio_cmd8_cont(cmd, parm);
    DisableIO();
    return res;
}

// Send 8-bit command + 16-bit parameter, de-assert, return response
uint16_t spi_uio_cmd16(uint8_t cmd, uint16_t parm) {
    spi_uio_cmd_cont(cmd);
    uint16_t res = spi_w(parm);
    DisableIO();
    return res;
}

// Send 8-bit command + 32-bit parameter (byte or word mode)
void spi_uio_cmd32(uint8_t cmd, uint32_t parm, int wide) {
    spi_uio_cmd_cont(cmd);
    if (wide) {
        spi_w((uint16_t)parm);
        spi_w(parm >> 16);
    } else {
        spi_b((uint8_t)parm);
        spi_b((uint8_t)(parm >> 8));
        spi_b((uint8_t)(parm >> 16));
        spi_b((uint8_t)(parm >> 24));
    }
    DisableIO();
}

// Send 8-bit command + 32-bit parameter as bytes, leave asserted
void spi_uio_cmd32_cont(uint8_t cmd, uint32_t parm) {
    spi_uio_cmd_cont(cmd);
    spi_b((uint8_t)parm);
    spi_b((uint8_t)(parm >> 8));
    spi_b((uint8_t)(parm >> 16));
    spi_b((uint8_t)(parm >> 24));
}
```

---

## 7. Transaction Patterns

### Pattern 1: Simple Command (No Response Data)

```
EnableIO()
  spi_w(UIO_COMMAND)
DisableIO()
```

### Pattern 2: Command + Response Word

```
EnableIO()
  response = spi_w(UIO_COMMAND)     // first word is the command, response comes simultaneously
  data = spi_w(0)                   // send zero to clock out response data
DisableIO()
```

### Pattern 3: Command + Multiple Parameters

```
EnableIO()
  spi_w(UIO_COMMAND)
  spi_w(param1)
  spi_w(param2)
  spi_w(param3)
DisableIO()
```

### Pattern 4: Command + Block Transfer

```
EnableIO()
  spi_w(UIO_COMMAND | ack_bits)
  spi_block_write(buffer, wide, block_size)   // or spi_block_read()
DisableIO()
```

### Pattern 5: OSD Line Write

```
EnableOsd()
  spi_b(0x20 | line_number)         // Write command + line
  spi_write(line_data, 256, 0)      // 256 bytes, 8-bit mode
DisableOsd()
```

### Pattern 6: FPGA File Transfer

```
EnableFpga()
  spi_b(FIO_FILE_TX)                // 0x53
  spi_b(0xFF)                       // enable download
DisableFpga()

EnableFpga()
  spi_b(FIO_FILE_TX_DAT)            // 0x54
  spi_write(data, length, fio_size) // file data
DisableFpga()

EnableFpga()
  spi_b(FIO_FILE_TX)                // 0x53
  spi_b(0x00)                       // disable (transfer complete)
DisableFpga()
```

---

## 8. Fast Block Transfer Implementation Details

The fast block transfer functions in `fpga_io.cpp` are manually unrolled 16 times for maximum throughput on the ARM CPU:

### Write (16-bit)

```c
void fpga_spi_fast_block_write(const uint16_t *buf, uint32_t length) {
    uint32_t gpo = fpga_gpo_read() & ~(0xFFFF | SSPI_STROBE);
    while (length >= 16) {
        // 16 iterations unrolled:
        fpga_gpo_writeN(gpo | SSPI_STROBE | *buf++);  // data + strobe high
        fpga_gpo_writeN(gpo | SSPI_STROBE | *buf++);
        // ... 14 more ...
        length -= 16;
    }
    while (length--) {
        fpga_gpo_writeN(gpo | SSPI_STROBE | *buf++);
    }
    fpga_gpo_write(gpo);  // update shadow copy at end
}
```

Key optimization: `fpga_gpo_writeN()` skips the shadow copy update inside the loop, only calling `fpga_gpo_write()` once at the end.

### Read (16-bit)

```c
void fpga_spi_fast_block_read(uint16_t *buf, uint32_t length) {
    uint32_t gpo = fpga_gpo_read() & ~(0xFFFF | SSPI_STROBE);
    while (length >= 16) {
        fpga_gpo_writeN(gpo | SSPI_STROBE);  // strobe high
        fpga_gpo_writeN(gpo);                  // strobe low
        *buf++ = fpga_gpi_read();              // read data
        // ... 15 more ...
        length -= 16;
    }
    // remainder loop
}
```

### 8-bit Variants

Same pattern but data placed in GPO[7:0] only, and GPI cast to `uint8_t`.

### Big-Endian Variants

Same as 16-bit but each word byte-swapped: `tmp = (tmp << 8) | (tmp >> 8)`.

---

## 9. Timing Considerations

### Handshake Mode

- Guaranteed reliable: FPGA explicitly acknowledges each word
- Latency per word: depends on FPGA processing time + memory-mapped I/O latency
- Typical: microseconds per word

### Fast Mode

- No guarantee: assumes FPGA responds within one GPO write + GPI read cycle
- The memory-mapped I/O through the AXI bus introduces enough latency that the FPGA has time to process
- Throughput: significantly higher than handshake mode
- Used for bulk data transfers (OSD lines, file data, disk sectors)

### Block Transfer Throughput

- 16x unrolling eliminates loop overhead
- `fpga_gpo_writeN()` skips shadow copy update (saves one memory write per iteration)
- ARM `volatile uint32_t*` writes are ordered and not coalesced by the compiler
- Actual throughput limited by AXI bus bandwidth (~100+ MHz memory-mapped I/O)

---

## 10. Error Handling

### FPGA Not Ready

During handshake SPI polling, if GPI bit 31 is set (FPGA not in user mode):
1. `fpga_spi()` calls `fpga_wait_to_reset()`
2. `fpga_wait_to_reset()` asserts core reset, polls `is_fpga_ready(0)` in 1-second intervals
3. Once FPGA is ready, performs warm reboot

### Timeout

The FPGA Manager polling functions use `FPGA_TIMEOUT_CNT` (16,777,216 iterations) as the maximum poll count. If exceeded, the programming function returns a negative error code:
- -1: reset phase timeout
- -2: config phase timeout
- -3: config error during programming
- -4: config done timeout
- -5: init phase DCLK error
- -6: init phase timeout
- -7: user mode DCLK error
- -8: user mode timeout

### Error Codes from `fpga_load_rbf()`

Returns 0 on success. Calls `reboot(1)` on programming failure.

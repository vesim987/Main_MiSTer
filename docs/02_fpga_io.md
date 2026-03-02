# FPGA I/O Layer (`fpga_io`) - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `fpga_io.h` | 44 | Public API header |
| `fpga_io.cpp` | 1010 | Full implementation |
| `fpga_base_addr_ac5.h` | 62 | Cyclone V SoC peripheral base addresses |
| `fpga_manager.h` | 70 | FPGA Manager register structure and constants |
| `fpga_reset_manager.h` | 74 | Reset Manager register structure |
| `fpga_system_manager.h` | 141 | System Manager register structure |
| `fpga_nic301.h` | 195 | NIC-301 L3 interconnect register structure |
| `shmem.h` | 13 | Shared memory mmap API |
| `shmem.cpp` | 73 | Shared memory mmap implementation |

---

## 1. Cyclone V SoC Base Address Map (`fpga_base_addr_ac5.h`)

Every peripheral base address used by the system:

```c
// Debug and Processing
SOCFPGA_DAP_ADDRESS             0xff000000   // Debug Access Port
SOCFPGA_MPUSCU_ADDRESS          0xfffec000   // MPU Snoop Control Unit
SOCFPGA_MPUL2_ADDRESS           0xfffef000   // MPU L2 Cache
SOCFPGA_OCRAM_ADDRESS           0xffff0000   // On-chip RAM
SOCFPGA_ROM_ADDRESS             0xfffd0000   // Boot ROM

// Network and Storage
SOCFPGA_EMAC0_ADDRESS           0xff700000   // Ethernet MAC 0
SOCFPGA_EMAC1_ADDRESS           0xff702000   // Ethernet MAC 1
SOCFPGA_SDMMC_ADDRESS           0xff704000   // SD/MMC Controller
SOCFPGA_QSPI_ADDRESS            0xff705000   // Quad SPI Flash Controller
SOCFPGA_NANDDATA_ADDRESS        0xff900000   // NAND Data
SOCFPGA_NANDREGS_ADDRESS        0xffb80000   // NAND Registers
SOCFPGA_QSPIDATA_ADDRESS        0xffa00000   // QSPI Data

// FPGA Manager
SOCFPGA_MGR_ADDRESS             0xff706000   // FPGA Manager (GPO/GPI registers)
SOCFPGA_FPGAMGRREGS_ADDRESS     0xff706000   // FPGA Manager Registers (same)
SOCFPGA_FPGAMGRDATA_ADDRESS     0xffb90000   // FPGA Manager Data (RBF write target)
SOCFPGA_ACPIDMAP_ADDRESS        0xff707000   // ACP ID Mapper

// FPGA Bridges
SOCFPGA_LWFPGASLAVES_ADDRESS    0xff200000   // Lightweight HPS-to-FPGA bridge (core registers)
SOCFPGA_LWHPS2FPGAREGS_ADDRESS  0xff400000   // LW HPS-to-FPGA bridge control registers
SOCFPGA_HPS2FPGAREGS_ADDRESS    0xff500000   // HPS-to-FPGA bridge control registers
SOCFPGA_FPGA2HPSREGS_ADDRESS    0xff600000   // FPGA-to-HPS bridge control registers

// GPIO
SOCFPGA_GPIO0_ADDRESS           0xff708000   // GPIO Bank 0
SOCFPGA_GPIO1_ADDRESS           0xff709000   // GPIO Bank 1
SOCFPGA_GPIO2_ADDRESS           0xff70a000   // GPIO Bank 2

// USB
SOCFPGA_USB0_ADDRESS            0xffb00000   // USB Controller 0
SOCFPGA_USB1_ADDRESS            0xffb40000   // USB Controller 1

// Communication
SOCFPGA_CAN0_ADDRESS            0xffc00000   // CAN Controller 0
SOCFPGA_CAN1_ADDRESS            0xffc01000   // CAN Controller 1
SOCFPGA_UART0_ADDRESS           0xffc02000   // UART 0
SOCFPGA_UART1_ADDRESS           0xffc03000   // UART 1
SOCFPGA_I2C0_ADDRESS            0xffc04000   // I2C 0
SOCFPGA_I2C1_ADDRESS            0xffc05000   // I2C 1
SOCFPGA_I2C2_ADDRESS            0xffc06000   // I2C 2
SOCFPGA_I2C3_ADDRESS            0xffc07000   // I2C 3

// SPI
SOCFPGA_SPIS0_ADDRESS           0xffe02000   // SPI Slave 0
SOCFPGA_SPIS1_ADDRESS           0xffe03000   // SPI Slave 1
SOCFPGA_SPIM0_ADDRESS           0xfff00000   // SPI Master 0
SOCFPGA_SPIM1_ADDRESS           0xfff01000   // SPI Master 1

// System
SOCFPGA_SDR_ADDRESS             0xffc20000   // SDRAM Controller
SOCFPGA_L3REGS_ADDRESS          0xff800000   // L3 Interconnect (NIC-301)
SOCFPGA_L4WD0_ADDRESS           0xffd02000   // L4 Watchdog 0
SOCFPGA_L4WD1_ADDRESS           0xffd03000   // L4 Watchdog 1
SOCFPGA_CLKMGR_ADDRESS          0xffd04000   // Clock Manager
SOCFPGA_RSTMGR_ADDRESS          0xffd05000   // Reset Manager
SOCFPGA_SYSMGR_ADDRESS          0xffd08000   // System Manager
SOCFPGA_SCANMGR_ADDRESS         0xfff02000   // Scan Manager

// Timers
SOCFPGA_SPTIMER0_ADDRESS        0xffc08000   // SP Timer 0
SOCFPGA_SPTIMER1_ADDRESS        0xffc09000   // SP Timer 1
SOCFPGA_OSC1TIMER0_ADDRESS      0xffd00000   // Oscillator Timer 0
SOCFPGA_OSC1TIMER1_ADDRESS      0xffd01000   // Oscillator Timer 1

// DMA
SOCFPGA_DMANONSECURE_ADDRESS    0xffe00000   // DMA Non-Secure
SOCFPGA_DMASECURE_ADDRESS       0xffe01000   // DMA Secure
```

---

## 2. Memory Mapping System

### Constants

```c
#define FPGA_REG_BASE    0xFF000000     // Physical base for the entire mmap region
#define FPGA_REG_SIZE    0x01000000     // 16 MB mapping size (0xFF000000 - 0xFFFFFFFF)
```

### Address Translation Macros

```c
// Convert physical address to pointer into mmap'd region.
// Masks off top byte (keeps lower 24 bits), divides by 4 for uint32_t indexing.
#define MAP_ADDR(x)  (volatile uint32_t*)(&map_base[(((uint32_t)(x)) & 0xFFFFFF)>>2])

// Test if address falls within FPGA register space
#define IS_REG(x)    (((((uint32_t)(x))-1)>=(FPGA_REG_BASE - 1)) && \
                      ((((uint32_t)(x))-1)<(FPGA_REG_BASE + FPGA_REG_SIZE - 1)))

// Register read/write
#define writel(val, reg)  *MAP_ADDR(reg) = val
#define readl(reg)        *MAP_ADDR(reg)

// Read-modify-write helpers
#define clrsetbits_le32(addr, clear, set)  writel((readl(addr) & ~(clear)) | (set), addr)
#define setbits_le32(addr, set)            writel( readl(addr) | (set), addr)
#define clrbits_le32(addr, clear)          writel( readl(addr) & ~(clear), addr)
```

### Initialization

```c
int fpga_io_init()
```

- Opens `/dev/mem` via `shmem_map(0xFF000000, 0x01000000)`
- Maps 16 MB of physical memory into user-space via `mmap()` with `PROT_READ | PROT_WRITE | MAP_SHARED`
- Stores mapped base pointer in `static uint32_t *map_base`
- Initializes GPO register to 0
- Returns -1 on failure, 0 on success

This single mapping covers ALL Cyclone V SoC peripherals from `0xFF000000` to `0xFFFFFFFF`.

---

## 3. GPO Register Bit Map (0xFF706010 - HPS to FPGA)

The central 32-bit register for HPS-to-FPGA communication:

```
Bit(s)   Usage
------   -----
[15:0]   SPI data output — 16-bit word sent to FPGA
[16]     (reserved)
[17]     SSPI_STROBE — SPI clock strobe. Toggled to clock data.
[18]     SSPI_FPGA_EN — Chip-select for FPGA core (file I/O channel)
[19]     SSPI_OSD_EN — Chip-select for OSD subsystem
[20]     SSPI_IO_EN — Chip-select for User I/O subsystem
[21:28]  (not explicitly assigned in code)
[29]     LED control — 1 = on, 0 = off
[30]     Core reset — set to 1 (with bit 31=0) to assert reset
[31]     Core active — set to 1 (with bit 30=0) to release reset / run mode
```

### GPO Access Functions

```c
// Write GPO register and update shadow copy
void fpga_gpo_write(uint32_t value);

// Fast GPO write without shadow copy update (for tight loops)
#define fpga_gpo_writeN(value)  writel((value), (void*)(SOCFPGA_MGR_ADDRESS + 0x10))

// Read GPO from shadow copy (not from hardware)
#define fpga_gpo_read()         gpo_copy
```

---

## 4. GPI Register Bit Map (0xFF706014 - FPGA to HPS)

The 32-bit response register from FPGA to HPS:

```
Bit(s)   Usage
------   -----
[15:0]   SPI data input — 16-bit word received from FPGA
         Also: core type ID (lower 8 bits) when reading fpga_core_id()
[16]     FIO size flag — 0 = 8-bit transfers, 1 = 16-bit transfers
[17]     SSPI_ACK — SPI acknowledge from FPGA (same position as SSPI_STROBE)
[19:18]  I/O version
[28]     I/O type flag
[30:29]  Button state — bit 29 = OSD button (BUTTON_OSD=1), bit 30 = USR button (BUTTON_USR=2)
[31]     FPGA ready flag — 0 = ready/user mode, 1 = uninitialized

Core ID validation: Upper 24 bits [31:8] must equal 0x5CA623 (magic number).
Lower 8 bits [7:0] = core type byte.
```

### GPI Access

```c
#define fpga_gpi_read()  (int)readl((void*)(SOCFPGA_MGR_ADDRESS + 0x14))
```

---

## 5. SPI Constants

```c
#define SSPI_STROBE   (1<<17)    // Bit 17 of GPO = strobe signal
#define SSPI_ACK      SSPI_STROBE // Bit 17 of GPI = acknowledge (same position)
#define SSPI_FPGA_EN  (1<<18)    // Bit 18 of GPO = FPGA chip select
#define SSPI_OSD_EN   (1<<19)    // Bit 19 of GPO = OSD chip select
#define SSPI_IO_EN    (1<<20)    // Bit 20 of GPO = User IO chip select
```

---

## 6. All Public Functions

### Core Control

```c
int fpga_core_id()
```
Reads core type from GPI register.
1. Clears GPO bit 31 (enters ID readback mode)
2. Reads GPI register
3. Restores GPO bit 31 to 1 (active mode)
4. Validates upper 24 bits == `0x5CA623`
5. Returns `coretype & 0xFF` on success, -1 on invalid magic

```c
void fpga_core_reset(int reset)
```
Controls core reset via GPO bits [31:30]:
- `reset=1`: Clears bits [31:30], then sets bit 30 (0x40000000) -- assert reset
- `reset=0`: Clears bits [31:30], then sets bit 31 (0x80000000) -- release reset (run mode)

```c
void fpga_set_led(uint32_t on)
```
Controls on-board LED via GPO bit 29.

```c
int fpga_get_buttons()
```
Reads GPI bits [30:29] for button state. Returns 0 if GPI is negative (FPGA not ready).
Return value: bit 0 = OSD button, bit 1 = USR button.

```c
int fpga_get_io_type()
```
Returns GPI bit 28.

```c
int fpga_get_fio_size()
```
Returns GPI bit 16: `(fpga_gpi_read() >> 16) & 1`. Determines 8-bit or 16-bit file I/O.

```c
int fpga_get_io_version()
```
Returns GPI bits [19:18]: `(fpga_gpi_read() >> 18) & 3`.

```c
int is_fpga_ready(int quick)
```
- Quick mode: returns `(fpga_gpi_read() >= 0)` -- just checks bit 31
- Full mode: calls `fpgamgr_test_fpga_ready()` which checks InitDone signal twice and verifies user mode

### Direct Register Access (Lightweight Bridge)

```c
void fpga_core_write(uint32_t offset, uint32_t value)
```
Writes `value` to the Lightweight HPS-to-FPGA bridge at `0xFF200000 + (offset & ~3)`.
Physical address: `SOCFPGA_LWFPGASLAVES_ADDRESS + offset` (forced to 4-byte alignment).
Only works for offsets up to `0x1FFFFF` (2 MB address space).

```c
uint32_t fpga_core_read(uint32_t offset)
```
Reads a 32-bit value from the Lightweight HPS-to-FPGA bridge.
Returns 0 for out-of-range offsets.

### Software SPI

```c
void fpga_spi_en(uint32_t mask, uint32_t en)
```
Enables or disables SPI chip-select lines.
Always sets bit 31 (active mode) in GPO.
If `en` is truthy, ORs `mask` into GPO. If `en` is falsy, clears `mask` bits.

```c
uint16_t fpga_spi(uint16_t word)
```
**Handshake SPI transfer.** Full protocol:
1. Place `word` in GPO[15:0], clear SSPI_STROBE
2. Set SSPI_STROBE high (bit 17)
3. Poll GPI until SSPI_ACK (bit 17) goes HIGH
4. Clear SSPI_STROBE
5. Poll GPI until SSPI_ACK goes LOW
6. Return GPI[15:0] as response
7. If GPI bit 31 set during polling: call `fpga_wait_to_reset()`, return 0

```c
uint16_t fpga_spi_fast(uint16_t word)
```
**No-handshake SPI transfer.** Faster but relies on timing:
1. Place `word` in GPO[15:0], clear SSPI_STROBE
2. Set SSPI_STROBE high
3. Immediately clear SSPI_STROBE (no ACK polling)
4. Return GPI[15:0] as response

### Block Transfers (Fast, Unrolled 16x)

```c
void fpga_spi_fast_block_write(const uint16_t *buf, uint32_t length)
```
Writes `length` 16-bit words via fast SPI. Loop unrolled 16x for performance.
Uses `fpga_gpo_writeN()` in inner loop (no shadow update), calls `fpga_gpo_write()` at end.

```c
void fpga_spi_fast_block_read(uint16_t *buf, uint32_t length)
```
Reads `length` 16-bit words via fast SPI. Loop unrolled 16x.

```c
void fpga_spi_fast_block_write_8(const uint8_t *buf, uint32_t length)
```
Writes `length` 8-bit bytes via fast SPI. Each byte placed in GPO[7:0].

```c
void fpga_spi_fast_block_read_8(uint8_t *buf, uint32_t length)
```
Reads `length` 8-bit bytes via fast SPI.

```c
void fpga_spi_fast_block_write_be(const uint16_t *buf, uint32_t length)
```
Writes `length` 16-bit words with byte-swap (big-endian conversion).

```c
void fpga_spi_fast_block_read_be(uint16_t *buf, uint32_t length)
```
Reads `length` 16-bit words with byte-swap (big-endian conversion).

### FPGA Programming

```c
int fpga_load_rbf(const char *name, const char *cfg, const char *xml)
```
Loads an RBF bitstream into the FPGA:
1. If `cfg` provided: reset core, write environment via `make_env()`, disable bridges, reboot
2. Open RBF file, read fully into memory
3. Detect MiSTer RBF format: first 6 bytes == "MiSTer", size at offset 12, data at offset 16
4. Disable bridges via `do_bridge(0)`
5. Call `socfpga_load(data, size)` to program FPGA
6. On success, enable bridges via `do_bridge(1)`
7. Call `app_restart()` to re-exec the process

### System Control

```c
void reboot(int cold)
```
Syncs filesystem, resets core. Maps shared memory at `0x1FFFF000`.
- Warm reboot (`cold=0`): writes magic `0xBEEFB001` at offset `0xF08`
- Cold reboot (`cold=1`): writes 0 at offset `0xF08`
- Triggers HPS reset by writing 1 to `reset_regs->ctrl`

```c
void app_restart(const char *path, const char *xml, const char *exe)
```
Syncs, resets core, destroys input devices, stops offload.
Calls `execl()` to replace current process. Falls back to `reboot(1)` if execl fails.

```c
char *getappname()
```
Returns executable path via `/proc/<pid>/exe` symlink.

```c
void fpga_wait_to_reset()
```
Called when FPGA detected as uninitialized. Polls `is_fpga_ready(0)` with 1-second sleeps, then warm reboots.

---

## 7. FPGA Programming Pipeline (Internal Functions)

### Programming Sequence

`socfpga_load()` orchestrates the full pipeline:

```
fpgamgr_program_init()        // Initialize FPGA for programming
    |
fpgamgr_program_write()       // Stream RBF data to FPGA Manager FIFO
    |
fpgamgr_program_poll_cd()     // Wait for CONF_DONE signal
    |
fpgamgr_program_poll_initphase()  // Wait for init phase
    |
fpgamgr_program_poll_usermode()   // Wait for user mode
```

### `fpgamgr_program_init()` Detail

1. Read MSEL pins from `fpgamgr_regs->stat` bits [7:3]
2. If MSEL[3]=1: 32-bit config width; determine CD ratio from MSEL[1:0]
3. If MSEL[3]=0: 16-bit config width; determine CD ratio from MSEL[1:0]
4. Enable FPGA Manager configuration (clear NCE bit)
5. Enable FPGA Manager drive (set EN bit)
6. Pull NCONFIG low (put FPGA into reset phase)
7. Wait for FPGA to enter reset phase (mode = 1)
8. Release NCONFIG (move FPGA out of reset)
9. Wait for FPGA to enter configuration phase (mode = 2)
10. Clear all interrupts in CB Monitor
11. Enable AXI configuration path

### `fpgamgr_program_write()` Detail

Uses ARM inline assembly for maximum throughput:
```asm
// Inner loop: load 32 bytes from source, store to single FIFO address
ldmia  %0!, {r0-r7}     // Load 8 registers (32 bytes) from source, auto-increment
stmia  %1,  {r0-r7}     // Store 8 registers to FIFO address (no auto-increment)
sub    %1, #32           // Reset destination (it's a FIFO, always same address)
```

Destination address: `SOCFPGA_FPGAMGRDATA_ADDRESS` (`0xFFB90000`)
Remaining bytes (< 32) handled with single `ldr`/`str` operations.

### `fpgamgr_program_poll_cd()` Detail

Waits for both `NS_MASK` (bit 0) and `CD_MASK` (bit 1) in `gpio_ext_porta`.
Disables AXI configuration on success.
Returns 0 on success, -3 on config error, -4 on timeout.

### `fpgamgr_program_poll_initphase()` Detail

Sends 4 additional DCLK cycles for CB initialization.
Waits for FPGA to enter init phase (mode 3) or user mode (mode 4).
Returns 0 on success, -5 on DCLK error, -6 on timeout.

### `fpgamgr_program_poll_usermode()` Detail

Sends 0x5000 (20,480) additional DCLK cycles.
Waits for FPGA to enter user mode (mode 4).
Releases FPGA Manager drive (clears EN bit).
Returns 0 on success, -7 on DCLK error, -8 on timeout.

---

## 8. Bridge Control (`do_bridge()`)

### Enable (enable=1)

```c
// Enable SDRAM controller FPGA ports (all 14 ports)
writel(0x00003FFF, SOCFPGA_SDR_ADDRESS + 0x5080);

// De-assert all bridge resets (bits 0,1,2 = lw-h2f, h2f, f2h)
writel(0x00000000, reset_regs->brg_mod_reset);

// Set L3 remap to make FPGA visible in address space
writel(0x00000019, nic301_regs->remap);
```

### Disable (enable=0)

```c
// Disable FPGA interface group in System Manager
writel(0, sysmgr_regs->fpgaintfgrp_module);

// Disable SDRAM FPGA ports
writel(0, SOCFPGA_SDR_ADDRESS + 0x5080);

// Assert all three bridge resets
writel(7, reset_regs->brg_mod_reset);

// Default remap (no FPGA visibility)
writel(1, nic301_regs->remap);
```

---

## 9. FPGA Manager Register Map (`fpga_manager.h`)

Structure `socfpga_fpga_manager` at base `0xFF706000`:

```
Offset   Field              Description
------   -----              -----------
0x00     stat               Status register (mode, MSEL)
0x04     ctrl               Control register (cfg width, CD ratio, NCE, EN, NCONFIGPULL, AXICFGEN)
0x08     dclkcnt            DCLK count register
0x0C     dclkstat           DCLK status register
0x10     gpo                General Purpose Output (HPS -> FPGA, 32 bits)
0x14     gpi                General Purpose Input (FPGA -> HPS, 32 bits)
0x18     misci              Miscellaneous input
0x830    gpio_inten         GPIO interrupt enable
0x834    gpio_intmask       GPIO interrupt mask
0x838    gpio_inttype_level GPIO interrupt type
0x83C    gpio_int_polarity  GPIO interrupt polarity
0x840    gpio_intstatus     GPIO interrupt status
0x844    gpio_raw_intstatus GPIO raw interrupt status
0x84C    gpio_porta_eoi     GPIO port A end-of-interrupt
0x850    gpio_ext_porta     GPIO external port A (monitoring: InitDone, nSTATUS, CONF_DONE, CRC)
0x860    gpio_1s_sync       GPIO 1s sync
0x86C    gpio_ver_id_code   GPIO version ID code
0x870    gpio_config_reg2   GPIO config register 2
0x874    gpio_config_reg1   GPIO config register 1
```

### Stat Register Masks

```c
FPGAMGRREGS_STAT_MODE_MASK    0x7     // bits [2:0] = current FPGA mode
FPGAMGRREGS_STAT_MSEL_MASK    0xf8    // bits [7:3] = MSEL pin values
FPGAMGRREGS_STAT_MSEL_LSB     3       // MSEL field starts at bit 3
```

### Ctrl Register Masks

```c
FPGAMGRREGS_CTRL_CFGWDTH_MASK     0x200  // bit 9 = config width (0=16-bit, 1=32-bit)
FPGAMGRREGS_CTRL_AXICFGEN_MASK    0x100  // bit 8 = AXI config enable
FPGAMGRREGS_CTRL_NCONFIGPULL_MASK 0x4    // bit 2 = NCONFIG pull (assert to enter reset)
FPGAMGRREGS_CTRL_NCE_MASK         0x2    // bit 1 = NCE (active low enable)
FPGAMGRREGS_CTRL_EN_MASK          0x1    // bit 0 = FPGA Manager enable
FPGAMGRREGS_CTRL_CDRATIO_LSB      6      // CD ratio field starts at bit 6
```

### GPIO External Port A Monitoring Masks (offset 0x850)

```c
FPGAMGRREGS_MON_GPIO_EXT_PORTA_CRC_MASK  0x8  // bit 3 = CRC error
FPGAMGRREGS_MON_GPIO_EXT_PORTA_ID_MASK   0x4  // bit 2 = Init Done
FPGAMGRREGS_MON_GPIO_EXT_PORTA_CD_MASK   0x2  // bit 1 = Config Done (CONF_DONE)
FPGAMGRREGS_MON_GPIO_EXT_PORTA_NS_MASK   0x1  // bit 0 = nSTATUS
```

### FPGA Modes

```c
FPGAMGRREGS_MODE_FPGAOFF      0x0   // FPGA is powered off
FPGAMGRREGS_MODE_RESETPHASE   0x1   // FPGA is in reset
FPGAMGRREGS_MODE_CFGPHASE     0x2   // FPGA is being configured
FPGAMGRREGS_MODE_INITPHASE    0x3   // FPGA initialization
FPGAMGRREGS_MODE_USERMODE     0x4   // FPGA is running user design
FPGAMGRREGS_MODE_UNKNOWN      0x5   // Unknown state
```

### Configuration-to-Data Clock Ratios

```c
CDRATIO_x1   0x0
CDRATIO_x2   0x1
CDRATIO_x4   0x2
CDRATIO_x8   0x3
```

---

## 10. Reset Manager Register Map (`fpga_reset_manager.h`)

Structure `socfpga_reset_manager` at base `0xFFD05000`:

```
Offset   Field             Description
------   -----             -----------
0x00     status            Reset status
0x04     ctrl              Reset control (writing 1 triggers warm reset)
0x08     counts            Reset counts
0x10     mpu_mod_reset     MPU module reset
0x14     per_mod_reset     Peripheral module reset
0x18     per2_mod_reset    Peripheral 2 module reset
0x1C     brg_mod_reset     Bridge module reset (bits 0/1/2 = lw-h2f/h2f/f2h)
0x20     misc_mod_reset    Miscellaneous module reset
0x54     tstscratch        Test scratch register
```

### Reset Identifiers

```c
RSTMGR_EMAC0        bank=1, offset=0     // Ethernet MAC 0
RSTMGR_EMAC1        bank=1, offset=1     // Ethernet MAC 1
RSTMGR_NAND         bank=1, offset=4
RSTMGR_QSPI         bank=1, offset=5
RSTMGR_L4WD0        bank=1, offset=6
RSTMGR_OSC1TIMER0   bank=1, offset=8
RSTMGR_UART0        bank=1, offset=16
RSTMGR_SPIM0        bank=1, offset=18
RSTMGR_SPIM1        bank=1, offset=19
RSTMGR_SDMMC        bank=1, offset=22
RSTMGR_DMA          bank=1, offset=28
RSTMGR_SDR          bank=1, offset=29
```

---

## 11. System Manager Register Map (`fpga_system_manager.h`)

Structure `socfpga_system_manager` at base `0xFFD08000`:

Key field used by fpga_io:

```
Offset   Field                  Description
------   -----                  -----------
0x28     fpgaintfgrp_module     FPGA interface group module enable
                                (written to 0 to disable FPGA interfaces during bridge disable)
```

### FPGA Interface Flags

```c
SYSMGR_FPGAINTF_USEFPGA  0x1
SYSMGR_FPGAINTF_SPIM0    (1 << 0)
SYSMGR_FPGAINTF_SPIM1    (1 << 1)
SYSMGR_FPGAINTF_EMAC0    (1 << 2)
SYSMGR_FPGAINTF_EMAC1    (1 << 3)
SYSMGR_FPGAINTF_NAND     (1 << 4)
SYSMGR_FPGAINTF_SDMMC    (1 << 5)
```

---

## 12. NIC-301 L3 Interconnect (`fpga_nic301.h`)

Structure `nic301_registers` at base `0xFF800000`:

Key field:

```
Offset   Field    Description
------   -----    -----------
0x00     remap    L3 remap register
                  0x00000019 = bridges enabled (FPGA visible in address space)
                  0x00000001 = default (FPGA not visible)
```

Also defines QoS registers for bus masters: DAP, MPU, SDMMC, DMA, FPGA2HPS, ETR, EMAC0, EMAC1, USB0, USB1, NAND, and write tidemarks for FPGAMGRDATA, HPS2FPGA, FPGA2HPS, OCRAM.

---

## 13. Shared Memory Helper (`shmem.h` / `shmem.cpp`)

```c
// Map physical memory into user-space via /dev/mem mmap
void *shmem_map(uint32_t address, uint32_t size);

// Unmap previously mapped memory
int shmem_unmap(void* map, uint32_t size);

// One-shot write: map, memcpy data to physical address, unmap
int shmem_put(uint32_t address, uint32_t size, void *buf);

// One-shot read: map, memcpy data from physical address, unmap
int shmem_get(uint32_t address, uint32_t size, void *buf);

// Convert FPGA SDRAM address to physical address
// FPGA sees SDRAM starting at 0x00000000; HPS sees it at 0x20000000
#define fpga_mem(x) (0x20000000 | ((x) & 0x1FFFFFFF))
```

### Implementation Details

`shmem_map()`:
1. Opens `/dev/mem` with `O_RDWR | O_SYNC | O_CLOEXEC`
2. Calls `mmap()` with `PROT_READ | PROT_WRITE | MAP_SHARED`
3. Closes the fd after mmap (the mapping persists)
4. Returns mapped pointer, or NULL on failure

---

## 14. Environment Block (`make_env()`)

Creates an environment block in shared memory at physical address `0x1FFFF000`:

```
Offset   Content
------   -------
0x00     Magic bytes: 0x21, 0x43, 0x65, 0x87
0x04     "core=<name>\n"
0x04+n   Contents of config file (if provided)
```

Used to pass core name and configuration to the next process instance after `app_restart()`.

---

## 15. Miscellaneous Constants

```c
#define FPGA_TIMEOUT_CNT   0x1000000    // 16,777,216 iterations for polling loops
#define DIV_ROUND_UP(n,d)  (((n) + (d) - 1) / (d))
#define BUTTON_OSD   1       // OSD button identifier
#define BUTTON_USR   2       // User button identifier
```

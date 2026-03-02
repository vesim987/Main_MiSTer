# Shared Memory & DMA Patterns - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `shmem.h` | 13 | API declarations |
| `shmem.cpp` | 73 | Implementation via `/dev/mem` mmap |
| `fpga_io.cpp` | 1010 | Uses shmem for register mapping, FPGA data, environment |
| `user_io.cpp` | 4177 | Uses shmem for save states, direct ROM loading |

---

## 1. Shared Memory API

```c
// Map physical memory into user-space via /dev/mem
void *shmem_map(uint32_t address, uint32_t size);

// Unmap previously mapped memory
int shmem_unmap(void* map, uint32_t size);

// One-shot write: map, memcpy to, unmap
int shmem_put(uint32_t address, uint32_t size, void *buf);

// One-shot read: map, memcpy from, unmap
int shmem_get(uint32_t address, uint32_t size, void *buf);

// Convert FPGA SDRAM address to HPS physical address
#define fpga_mem(x) (0x20000000 | ((x) & 0x1FFFFFFF))
```

### Implementation

`shmem_map()`:
1. Opens `/dev/mem` with `O_RDWR | O_SYNC | O_CLOEXEC`
2. Calls `mmap()` with `PROT_READ | PROT_WRITE | MAP_SHARED`
3. Closes the fd (mapping persists after close)
4. Returns mapped pointer, or NULL on failure

`shmem_unmap()`:
1. Calls `munmap()` on the pointer

`shmem_put()` / `shmem_get()`:
1. Map the region
2. `memcpy()` data to/from the mapped region
3. Unmap

---

## 2. Physical Memory Map

### Hardware Registers (0xFF000000 - 0xFFFFFFFF)

Mapped once at startup by `fpga_io_init()`:

```c
map_base = shmem_map(0xFF000000, 0x01000000);  // 16 MB
```

All hardware register access goes through this single mapping using `MAP_ADDR()` macro.

### FPGA SDRAM (0x20000000+)

The Cyclone V SoC maps FPGA-accessible SDRAM starting at physical address `0x20000000` (512 MB mark). The `fpga_mem()` macro translates:

```c
#define fpga_mem(x) (0x20000000 | ((x) & 0x1FFFFFFF))
```

- FPGA sees SDRAM addresses starting at `0x00000000`
- HPS sees the same SDRAM at `0x20000000`
- The macro ensures the address falls within the lower 512 MB

### Reboot/Environment Page (0x1FFFF000)

A single 4 KB page at `0x1FFFF000` used for persistent data across reboots/process restarts:

```
Offset    Size    Content
------    ----    -------
0xF00     4       SDRAM configuration
                  Bytes: 0x12, 0x57, then 16-bit config value

0xF04     5       Alternate config index
                  Bytes: 0x34, 0x99, 0xBA, then 16-bit index

0xF08     4       Reboot magic
                  0xBEEFB001 = warm reboot
                  0x00000000 = cold reboot
```

### Environment Block (0x1FFFF000, offset 0x00)

Written by `make_env()` before app_restart:

```
Offset    Content
------    -------
0x00      Magic: 0x21, 0x43, 0x65, 0x87
0x04      "core=<name>\n"
0x04+n    Contents of config file (optional)
```

---

## 3. Memory Usage Patterns

### Pattern 1: Hardware Register Access

The entire 16 MB register space is mapped once at startup:

```c
// In fpga_io_init():
map_base = (uint32_t *)shmem_map(FPGA_REG_BASE, FPGA_REG_SIZE);

// All subsequent register access:
#define MAP_ADDR(x) (volatile uint32_t*)(&map_base[(((uint32_t)(x)) & 0xFFFFFF)>>2])
#define writel(val, reg) *MAP_ADDR(reg) = val
#define readl(reg)       *MAP_ADDR(reg)
```

This covers:
- FPGA Manager GPO/GPI (software SPI)
- Lightweight HPS-to-FPGA bridge (core registers)
- Reset Manager, System Manager, NIC-301
- FPGA Manager data port (bitstream FIFO)

### Pattern 2: FPGA Bitstream Programming

In `fpgamgr_program_write()`, the FPGA Manager data FIFO at `0xFFB90000` is accessed through the same mapping. ARM assembly burst writes 32 bytes at a time:

```c
// Physical address 0xFFB90000 mapped through map_base
volatile uint32_t *dest = MAP_ADDR(SOCFPGA_FPGAMGRDATA_ADDRESS);

// ARM assembly for burst write:
asm volatile (
    "ldmia %0!, {r0-r7}\n"    // Load 32 bytes from source
    "stmia %1,  {r0-r7}\n"    // Store to FIFO (same address)
    "sub   %1,  #32\n"        // Reset destination (FIFO doesn't auto-increment)
    : "+r"(src), "+r"(dest)
    : : "r0","r1","r2","r3","r4","r5","r6","r7"
);
```

### Pattern 3: Direct ROM/Data Loading

For large data transfers (ROM files, disk images), data is written directly to FPGA-accessible SDRAM, completely bypassing the SPI channel:

```c
// In user_io_file_tx() when load_addr >= 0x20000000:
uint32_t phys_addr = fpga_mem(load_addr);  // e.g., 0x20000000 | addr
uint8_t *mem = (uint8_t *)shmem_map(phys_addr, map_size);

// Read file data directly into FPGA SDRAM:
while (remaining > 0) {
    int chunk = FileReadAdv(&file, mem + offset, chunk_size);
    offset += chunk;
    remaining -= chunk;
}

shmem_unmap(mem, map_size);
```

This is **significantly faster** than transferring data word-by-word over the software SPI protocol.

### Pattern 4: Save States

Save states use shared SDRAM at an address specified by the core's config string:

```c
// Config string: "SS[base_address],[slot_size]"
// e.g., "SS0x10000000,0x100000" = base at 256MB, 1MB per slot

// Map 4 slots:
uint32_t total_size = ss_size * 4;
uint8_t *ss_mem = (uint8_t *)shmem_map(fpga_mem(ss_base), total_size);

// For each slot:
for (int slot = 0; slot < 4; slot++) {
    uint8_t *slot_ptr = ss_mem + (slot * ss_size);
    uint32_t counter = *(uint32_t *)slot_ptr;

    if (counter != cached_counter[slot]) {
        // FPGA updated this slot — save to disk
        write_to_file(slot_ptr, ss_size, filename);
        cached_counter[slot] = counter;
    }
}

shmem_unmap(ss_mem, total_size);
```

The FPGA core increments a counter at offset 0 of each slot when it writes new save state data. The HPS polls every 1 second and writes changed data to disk.

### Pattern 5: Reboot/Config Persistence

Small values persisted across warm reboots via the shared memory page:

```c
// Write reboot magic for warm reboot:
uint32_t *magic = (uint32_t *)shmem_map(0x1FFFF000, 0x1000);
magic[0xF08/4] = 0xBEEFB001;  // warm reboot
shmem_unmap(magic, 0x1000);

// Write SDRAM config:
void sdram_sz_write(uint16_t sz) {
    uint8_t buf[4] = {0x12, 0x57, (uint8_t)sz, (uint8_t)(sz >> 8)};
    shmem_put(0x1FFFF000 + 0xF00, 4, buf);
}

// Write alt config index:
void altcfg_write(uint16_t alt) {
    uint8_t buf[5] = {0x34, 0x99, 0xBA, (uint8_t)alt, (uint8_t)(alt >> 8)};
    shmem_put(0x1FFFF000 + 0xF04, 5, buf);
}
```

### Pattern 6: One-Shot Memory Access

For small, infrequent accesses (reading a few bytes from FPGA memory):

```c
// Read a value from FPGA SDRAM:
uint32_t value;
shmem_get(fpga_mem(address), sizeof(uint32_t), &value);

// Write a value to FPGA SDRAM:
uint32_t data = 0x12345678;
shmem_put(fpga_mem(address), sizeof(uint32_t), &data);
```

These map-copy-unmap in a single call. Useful when the overhead of maintaining a persistent mapping isn't justified.

---

## 4. DMA-Like Mechanisms

There is **no hardware DMA controller** used between HPS and FPGA in this codebase. Instead, several patterns serve as DMA substitutes:

### CPU Burst Writes (FPGA Programming)

ARM `ldmia`/`stmia` instructions write 32 bytes per iteration to the FPGA Manager FIFO. This is the closest to hardware DMA — it saturates the AXI bus bandwidth using the CPU.

### Direct SDRAM Access (ROM Loading)

Memory-mapped access to FPGA SDRAM via `shmem_map(fpga_mem(addr), ...)`. The CPU reads file data and writes it directly to SDRAM that the FPGA can access. No SPI overhead.

### SPI-Based DMA Commands

Some cores (x86/ao486, IDE) use SPI commands to trigger DMA-like transfers within the FPGA fabric:

```c
UIO_DMA_WRITE  0x61  // HPS sends data to FPGA via SPI block transfer
UIO_DMA_READ   0x62  // FPGA sends data to HPS via SPI block transfer
UIO_DMA_SDIO   0x63  // SDIO DMA transfer
```

After sending the command, data is streamed using `spi_block_write()` or `spi_block_read()` with the fast (non-handshake) SPI mode.

---

## 5. Communication Channel Comparison

| Channel | Throughput | Latency | Use Case |
|---------|-----------|---------|----------|
| Software SPI (handshake) | Low (~100KB/s) | High (ACK polling) | Commands, config, small data |
| Software SPI (fast) | Medium (~1MB/s) | Low (no ACK) | OSD lines, disk sectors |
| LW HPS-to-FPGA bridge | High (~10MB/s) | Very low (memory-mapped) | Core register read/write |
| Shared SDRAM | Very high (~100MB/s) | Very low (memory-mapped) | ROM loading, save states |
| FPGA Manager FIFO | High (~50MB/s) | Low (burst writes) | Bitstream programming only |

### When to Use Each

- **SPI (handshake):** Reliable command/response — use for configuration, status, small transfers
- **SPI (fast):** Bulk data where some error tolerance exists — OSD, disk sector I/O
- **LW bridge:** Direct register access to core-specific addresses — control registers, status polling
- **Shared SDRAM:** Large data loads — ROMs (multi-MB), save states, disk images
- **FPGA Manager:** Only during FPGA programming — never used at runtime

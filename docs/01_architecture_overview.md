# MiSTer HPS-FPGA Communication Architecture Overview

## Platform

**Hardware:** Intel/Altera Cyclone V SoC FPGA on the Terasic DE10-Nano board.

The Cyclone V SoC integrates:
- **HPS (Hard Processor System):** Dual-core ARM Cortex-A9 running Linux
- **FPGA Fabric:** Cyclone V programmable logic (loads retro computing/gaming cores)

The `MiSTer` binary runs on the HPS ARM cores and manages all communication with whatever FPGA core is currently loaded.

## Process Model

- Single binary named `MiSTer`, cross-compiled with `arm-none-linux-gnueabihf-gcc` (GCC 10.2.1)
- Pinned to CPU core #1 via `sched_setaffinity()`
- Background offload worker thread on CPU core #0 (`offload.cpp`)
- Cooperative multitasking via libco coroutines (`scheduler.cpp`):
  - `co_poll`: runs `user_io_poll()` + `input_poll()`
  - `co_ui`: runs `HandleUI()` + `OsdUpdate()`

## Communication Channels

There are **five distinct communication channels** between the HPS and FPGA:

| # | Channel | Physical Address | Size | Purpose |
|---|---------|-----------------|------|---------|
| 1 | GPO Register | `0xFF706010` | 4 bytes | HPS->FPGA: software SPI data + control signals |
| 2 | GPI Register | `0xFF706014` | 4 bytes | FPGA->HPS: software SPI response + status |
| 3 | LW HPS-to-FPGA Bridge | `0xFF200000` | 2 MB | Direct 32-bit register read/write into core address space |
| 4 | Shared SDRAM | `0x20000000+` | up to 512 MB | Bulk data transfers (ROM loading, save states) |
| 5 | FPGA Manager Data | `0xFFB90000` | FIFO | RBF bitstream programming (write-only) |

### Channel Details

**1. GPO/GPI (Software SPI):** The primary command/control channel. A 16-bit bidirectional, bit-banged SPI protocol using the FPGA Manager's General Purpose Output and Input registers. Supports both handshake (reliable) and fast (high-throughput) modes. Three chip-select lines multiplex access to different FPGA subsystems: core data (FPGA_EN), OSD display (OSD_EN), and user I/O (IO_EN).

**2. Lightweight HPS-to-FPGA Bridge:** A 2 MB memory window at `0xFF200000`-`0xFF3FFFFF`. Provides direct 32-bit register access to core-defined addresses. Used by `fpga_core_read()` and `fpga_core_write()`. Offset must be 4-byte aligned and <= `0x1FFFFF`.

**3. Shared SDRAM:** Physical memory starting at `0x20000000` (512 MB mark) accessible by both HPS and FPGA via the SDRAM controller's FPGA port. Used for bulk ROM/data loading, bypassing the SPI channel entirely. Accessed via `shmem_map()` which mmaps `/dev/mem`.

**4. FPGA Manager Data Port:** A write-only FIFO at `0xFFB90000` used exclusively during FPGA programming to stream the RBF (Raw Binary File) bitstream. Uses ARM `ldmia`/`stmia` assembly for burst writes (32 bytes per iteration).

**5. Bridge Control:** The three HPS-FPGA bridges (LW-HPS2FPGA, HPS2FPGA, FPGA2HPS) must be explicitly enabled after FPGA programming via `do_bridge(1)`, which configures the Reset Manager, NIC-301 L3 remap register, and SDRAM controller FPGA ports.

## Core Types

Each FPGA core reports its type via a magic+type ID read from the GPI register:

| Constant | Value | Description |
|----------|-------|-------------|
| `CORE_TYPE_UNKNOWN` | `0x55` | Invalid/broken core (unlikely random value) |
| `CORE_TYPE_8BIT` | `0xa4` | Generic MiSTer core (vast majority of cores) |
| `CORE_TYPE_SHARPMZ` | `0xa7` | Sharp MZ Series core |
| `CORE_TYPE_8BIT2` | `0xa8` | Dual SDRAM core (remapped to 8BIT internally, sets `dual_sdr=1`) |

The core type is validated by checking that the upper 24 bits of the GPI register equal the magic value `0x5CA623`. The lower 8 bits contain the core type byte.

## Memory Map Overview

All hardware access is through a single 16 MB mmap of `/dev/mem` at physical base `0xFF000000`:

```
0xFF000000 - 0xFF1FFFFF   (various SoC peripherals - DAP, timers, etc.)
0xFF200000 - 0xFF3FFFFF   Lightweight HPS-to-FPGA Bridge (core registers)
0xFF400000 - 0xFF4FFFFF   LW HPS-to-FPGA Bridge control registers
0xFF500000 - 0xFF5FFFFF   HPS-to-FPGA Bridge control registers
0xFF600000 - 0xFF6FFFFF   FPGA-to-HPS Bridge control registers
0xFF700000 - 0xFF70FFFF   Ethernet, SD/MMC, QSPI, FPGA Manager
  0xFF706000               FPGA Manager base
  0xFF706010               GPO register (HPS->FPGA)
  0xFF706014               GPI register (FPGA->HPS)
0xFF800000                 NIC-301 L3 Interconnect (remap register)
0xFFB00000 - 0xFFBFFFFF   USB controllers, NAND, FPGA Manager data
  0xFFB90000               FPGA Manager data FIFO (bitstream target)
0xFFC00000 - 0xFFC0FFFF   CAN, UART, I2C, SPI, timers
0xFFD00000 - 0xFFD0FFFF   Timers, Watchdogs, Clock Manager, Reset Manager, System Manager
  0xFFD05000               Reset Manager
  0xFFD08000               System Manager
0xFFF00000 - 0xFFFEFFFF   SPI Masters, Scan Manager, Boot ROM, MPU SCU, L2 Cache, OCRAM
```

Additionally, FPGA-accessible SDRAM is mapped separately as needed:
```
0x20000000 - 0x3FFFFFFF   FPGA SDRAM window (accessed via fpga_mem() macro)
0x1FFFF000 - 0x1FFFFFFF   Reboot/environment shared memory page
```

## Software Architecture Layers

```
+--------------------------------------------------+
|  Menu System (menu.cpp)                          |  UI state machine, ~100 states
|  - HandleUI()                                    |  Config string parsing
|  - File browser                                  |  Settings management
+--------------------------------------------------+
|  OSD Renderer (osd.cpp)                          |  256x32 byte framebuffer
|  - OsdWrite(), OsdUpdate()                       |  8x8 font rendering
|  - Title sidestripe, arrows, scrolling           |  Starfield animation
+--------------------------------------------------+
|  User I/O Protocol (user_io.cpp)                 |  Core communication
|  - Keyboard, mouse, joystick forwarding          |  SD card emulation
|  - File transfers, status management             |  PS/2 protocol
|  - Core init/poll loop                           |  RTC, UART, video config
+--------------------------------------------------+
|  SPI Protocol Layer (spi.cpp)                    |  Chip-select management
|  - spi_uio_cmd*(), spi_osd_cmd*()               |  Block transfers
|  - EnableFpga/IO/Osd, DisableFpga/IO/Osd        |  OSD target routing
+--------------------------------------------------+
|  FPGA I/O (fpga_io.cpp)                          |  Bit-banged SPI engine
|  - fpga_spi(), fpga_spi_fast()                   |  GPO/GPI register access
|  - fpga_core_read/write()                        |  LW bridge access
|  - fpga_load_rbf()                               |  FPGA programming
|  - do_bridge()                                   |  Bridge enable/disable
+--------------------------------------------------+
|  Shared Memory (shmem.cpp)                       |  /dev/mem mmap
|  - shmem_map(), shmem_unmap()                    |  Physical address access
|  - fpga_mem() macro                              |  SDRAM translation
+--------------------------------------------------+
|  Hardware (Cyclone V SoC)                        |
|  - FPGA Manager registers                        |
|  - Reset Manager, System Manager, NIC-301        |
|  - HPS-FPGA bridges, SDRAM controller            |
+--------------------------------------------------+
```

## Boot Sequence

1. `main()` calls `FindStorage()` to locate SD card or USB storage
2. `fpga_io_init()` mmaps the 16 MB hardware register space
3. `fpga_load_rbf()` loads the MENU core (or last-used core via `bootcore.cpp`)
4. `user_io_init()` initializes the core: reads core type, config string, loads ROMs/VHDs, configures video/audio
5. `scheduler_run()` starts the cooperative scheduler loop:
   - `co_poll` coroutine: `user_io_poll()` + `input_poll()` (core communication + input processing)
   - `co_ui` coroutine: `HandleUI()` + `OsdUpdate()` (menu state machine + OSD flush)

When the user selects a new core from the menu:
1. `fpga_load_rbf()` is called with the new RBF path
2. Bridges are disabled (`do_bridge(0)`)
3. FPGA is reprogrammed via the FPGA Manager hardware
4. Bridges are re-enabled (`do_bridge(1)`)
5. `app_restart()` calls `execl()` to re-exec the process with the new core path

## Project Structure

```
Main_MiSTer/
|-- main.cpp                  Entry point, main loop
|-- fpga_io.cpp/.h            FPGA I/O: bitstream loading, SPI, bridges
|-- spi.cpp/.h                SPI protocol layer
|-- user_io.cpp/.h            Core communication protocol
|-- osd.cpp/.h                On-screen display rendering
|-- menu.cpp/.h               Menu state machine (7343 lines)
|-- input.cpp/.h              Input device management
|-- video.cpp/.h              Video output, framebuffer, scaler
|-- audio.cpp/.h              Audio output
|-- file_io.cpp/.h            File I/O with ZIP support
|-- cfg.cpp/.h                INI configuration parser
|-- shmem.cpp/.h              Shared memory mmap helper
|-- charrom.cpp/.h            OSD font data
|-- scheduler.cpp/.h          Cooperative scheduler (libco)
|-- offload.cpp/.h            Background worker thread
|-- bootcore.cpp/.h           Boot core selection
|-- DiskImage.cpp/.h          Disk image abstraction
|-- ide.cpp/.h                IDE/ATA emulation
|-- cheats.cpp/.h             Cheat code support
|-- battery.cpp/.h            Battery-backed SRAM saves
|-- hardware.cpp/.h           Hardware detection
|-- fpga_manager.h            FPGA Manager register map
|-- fpga_base_addr_ac5.h      Cyclone V SoC base addresses
|-- fpga_reset_manager.h      Reset Manager register map
|-- fpga_system_manager.h     System Manager register map
|-- fpga_nic301.h             NIC-301 interconnect register map
|-- support/                  Per-core support modules (16 platforms)
|   |-- arcade/               MRA arcade ROM loader
|   |-- archie/               Acorn Archimedes
|   |-- c64/                  Commodore 64
|   |-- chd/                  CHD disc image support
|   |-- megacd/               Sega Mega CD
|   |-- minimig/              Amiga (Minimig)
|   |-- n64/                  Nintendo 64
|   |-- neogeo/               SNK Neo Geo
|   |-- pcecd/                PC Engine CD
|   |-- psx/                  Sony PlayStation
|   |-- saturn/               Sega Saturn
|   |-- sharpmz/              Sharp MZ Series
|   |-- snes/                 Super Nintendo
|   |-- st/                   Atari ST
|   |-- uef/                  BBC Micro UEF
|   `-- x86/                  ao486/PCXT
|-- lib/                      Third-party libraries
|   |-- bluetooth/            Bluetooth/HCI
|   |-- imlib2/               Image library
|   |-- libchdr/              CHD disc image library
|   |-- libco/                Cooperative threading
|   |-- lzma/                 LZMA compression
|   |-- md5/                  MD5 hashing
|   |-- miniz/                Lightweight zlib replacement
|   `-- zstd/                 Zstandard decompression
`-- Makefile                  Build system (ARM cross-compilation)
```

## Build System

- **Toolchain:** `arm-none-linux-gnueabihf-gcc` (GCC 10.2.1)
- **C standard:** gnu99; **C++ standard:** gnu++14
- **Optimization:** `-O3`
- **Output:** `MiSTer` binary (stripped), `MiSTer.elf` (debug symbols)
- **Linked libraries:** `-lc -lstdc++ -lm -lrt -lfreetype -lbz2 -lpng16 -lz -lImlib2 -lbluetooth -lpthread`
- **Source patterns:** `*.cpp`, `*.c`, `lib/*/*.c`, `support/*/*.cpp`, `lib/libco/arm.c`

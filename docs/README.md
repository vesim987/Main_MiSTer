# MiSTer Main_MiSTer Technical Documentation

Comprehensive technical reference for the MiSTer FPGA HPS (Hard Processor System) software. This documentation covers every register, protocol, command byte, function signature, and data structure needed to understand or reimplement the HPS-FPGA communication stack from scratch.

## Document Index

| # | Document | Description |
|---|----------|-------------|
| 1 | [Architecture Overview](01_architecture_overview.md) | Platform, communication channels, memory map, software layers, boot sequence, project structure |
| 2 | [FPGA I/O](02_fpga_io.md) | Low-level hardware layer: register maps, memory mapping, GPO/GPI bit assignments, FPGA programming pipeline, bridge control |
| 3 | [SPI Protocol](03_spi_protocol.md) | Software SPI bit-bang protocol: handshake/fast modes, chip selects, block transfers, OSD/UIO command wrappers |
| 4 | [User I/O](04_user_io.md) | Core communication protocol: complete SPI command table (60+ commands), keyboard/mouse/joystick formats, SD card emulation, file transfer, status management |
| 5 | [OSD](05_osd.md) | On-Screen Display: rendering pipeline, memory layout, font system, SPI protocol, visual elements, control characters |
| 6 | [Menu System](06_menu_system.md) | State machine (~100 states), config string to menu mapping, file browser, settings persistence, MGL scripting |
| 7 | [Core Support](07_core_support.md) | Core lifecycle, scheduler, offload worker, boot core selection, 16 platform-specific support modules, save states |
| 8 | [File I/O](08_file_io.md) | File operations with transparent ZIP support, directory scanning, config file I/O, storage system |
| 9 | [Configuration](09_configuration.md) | INI parser, cfg_t structure, per-core sections, 128-bit status, SDRAM/alt-config persistence, UART config |
| 10 | [Shared Memory & DMA](10_shared_memory.md) | Physical memory mapping, FPGA SDRAM access, save state mechanism, DMA-like patterns, channel comparison |

## Quick Reference

### Communication Channels

| Channel | Address | Throughput | Use Case |
|---------|---------|-----------|----------|
| GPO/GPI (Software SPI) | `0xFF706010`/`0xFF706014` | ~1 MB/s | Commands, config, small data |
| LW HPS-to-FPGA Bridge | `0xFF200000` | ~10 MB/s | Core register R/W |
| Shared SDRAM | `0x20000000+` | ~100 MB/s | ROM loading, save states |
| FPGA Manager Data | `0xFFB90000` | ~50 MB/s | Bitstream programming only |

### SPI Chip Selects

| Line | GPO Bit | Enable/Disable | Purpose |
|------|---------|----------------|---------|
| FPGA | 18 | `EnableFpga()`/`DisableFpga()` | File I/O (FIO_* commands) |
| OSD | 19 (+18,20) | `EnableOsd()`/`DisableOsd()` | OSD display data |
| IO | 20 | `EnableIO()`/`DisableIO()` | User I/O (UIO_* commands) |

### Core Types

| Type | Value | Description |
|------|-------|-------------|
| `CORE_TYPE_8BIT` | `0xa4` | Generic MiSTer core |
| `CORE_TYPE_SHARPMZ` | `0xa7` | Sharp MZ series |
| `CORE_TYPE_8BIT2` | `0xa8` | Dual SDRAM (mapped to 8BIT) |

### Key SPI Commands

| Command | Value | Channel | Purpose |
|---------|-------|---------|---------|
| `UIO_GET_STRING` | `0x14` | IO | Read config string from core |
| `UIO_SET_STATUS2` | `0x1E` | IO | Send 128-bit status to core |
| `UIO_GET_STATUS` | `0x29` | IO | Detect status changes |
| `UIO_KEYBOARD` | `0x05` | IO | Send keyboard scancode |
| `UIO_MOUSE` | `0x04` | IO | Send mouse data |
| `UIO_JOYSTICK0` | `0x02` | IO | Send joystick data |
| `UIO_GET_SDSTAT` | `0x16` | IO | Poll SD card emulation |
| `FIO_FILE_TX` | `0x53` | FPGA | File transfer control |
| `FIO_FILE_TX_DAT` | `0x54` | FPGA | File transfer data |
| `OSD_CMD_WRITE` | `0x20` | OSD | Write OSD line data |
| `OSD_CMD_ENABLE` | `0x41` | OSD | Enable OSD |
| `OSD_CMD_DISABLE` | `0x40` | OSD | Disable OSD |

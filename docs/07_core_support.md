# Core Support Infrastructure - Complete Reference

## Source Files

### Core Infrastructure

| File | Lines | Role |
|------|-------|------|
| `main.cpp` | ~100 | Application entry point, main loop |
| `user_io.cpp` | 4177 | Core communication protocol |
| `fpga_io.cpp` | 1010 | FPGA programming, SPI, bridges |
| `scheduler.cpp` | 93 | Cooperative scheduler (libco) |
| `offload.cpp` | 122 | Background worker thread |
| `bootcore.cpp` | 235 | Boot core selection logic |
| `support.h` | — | Umbrella header for all support modules |

### Per-Core Support Modules (16 platforms)

| Directory | Platform | Source Files |
|-----------|----------|-------------|
| `support/arcade/` | MRA Arcade | `mra_loader.cpp`, `buffer.cpp` |
| `support/archie/` | Acorn Archimedes | `archie.cpp` |
| `support/c64/` | Commodore 64 | `c64.cpp` |
| `support/chd/` | CHD disc images | `mister_chd.cpp` |
| `support/megacd/` | Sega Mega CD | `megacd.cpp`, `megacdd.cpp` |
| `support/minimig/` | Amiga (Minimig) | `minimig_config.cpp`, `minimig_boot.cpp`, `minimig_fdd.cpp`, `minimig_share.cpp` |
| `support/n64/` | Nintendo 64 | `n64.cpp`, `n64_joy_emu.cpp` |
| `support/neogeo/` | SNK Neo Geo | `neogeo_loader.cpp`, `neogeocd.cpp` |
| `support/pcecd/` | PC Engine CD | `pcecd.cpp`, `pcecdd.cpp`, `seektime.cpp` |
| `support/psx/` | Sony PlayStation | `psx.cpp` |
| `support/saturn/` | Sega Saturn | `saturn.cpp`, `saturncdd.cpp` |
| `support/sharpmz/` | Sharp MZ Series | `sharpmz.cpp` |
| `support/snes/` | Super Nintendo | `snes.cpp` |
| `support/st/` | Atari ST | `st_tos.cpp` |
| `support/uef/` | BBC Micro (Electron) | `uef_reader.cpp` |
| `support/x86/` | ao486/PCXT | `x86.cpp`, `x86_share.cpp` |

---

## 1. Application Entry Point (`main.cpp`)

```c
int main(int argc, char *argv[]) {
    // 1. Pin process to CPU core #1
    sched_setaffinity(0, sizeof(cpu_set_t), &cpu_set);  // core 1

    // 2. Find storage (SD card / USB)
    FindStorage();

    // 3. Initialize FPGA I/O (mmap 16MB hardware registers)
    fpga_io_init();

    // 4. Load FPGA core
    //    - If argv has a core path: load that core
    //    - Otherwise: load MENU core, then check bootcore/lastcore
    fpga_load_rbf(core_path, cfg_path, xml_path);

    // 5. Initialize core communication
    user_io_init(core_path, xml_path);

    // 6. Enter scheduler loop (never returns normally)
    scheduler_run();
}
```

---

## 2. Scheduler (`scheduler.cpp`)

Uses **libco** (cooperative threading library, ARM backend via `lib/libco/arm.c`).

Two coroutines alternate execution:

```c
static cothread_t co_main;    // Main thread context
static cothread_t co_poll;    // Poll coroutine
static cothread_t co_ui;      // UI coroutine

void scheduler_run() {
    co_main = co_active();
    co_poll = co_create(65536, poll_thread);   // 64KB stack
    co_ui = co_create(65536, ui_thread);       // 64KB stack

    while (1) {
        co_switch(co_poll);    // Run poll_thread until it yields
        co_switch(co_ui);      // Run ui_thread until it yields
    }
}

static void poll_thread(void) {
    while (1) {
        user_io_poll();        // Core communication, SD card emulation
        input_poll();          // Input device polling
        co_switch(co_main);    // Yield back to scheduler
    }
}

static void ui_thread(void) {
    while (1) {
        HandleUI();            // Menu state machine
        OsdUpdate();           // Flush OSD dirty lines to FPGA
        co_switch(co_main);    // Yield back to scheduler
    }
}
```

---

## 3. Offload Worker (`offload.cpp`)

Background thread pinned to CPU core #0:

```c
#define OFFLOAD_QUEUE_SIZE 8

struct offload_entry {
    void (*func)(void*);
    void* param;
};

static offload_entry queue[OFFLOAD_QUEUE_SIZE];
static int head = 0, tail = 0;
static pthread_t worker_thread;
static pthread_mutex_t mutex;
static pthread_cond_t cond;

void offload_start(void (*func)(void*), void* param);  // Queue a work item
void offload_stop();                                      // Wait for all items to complete
```

The worker thread runs on core #0, processing queued function pointers. Used for background file operations that would otherwise block the main loop.

---

## 4. Boot Core Selection (`bootcore.cpp`)

### Last Core Persistence

- `lastcore.dat`: saved at `config/lastcore.dat`
- Contains the path of the last loaded core
- On boot, if `bootcore_timeout > 0` in INI, waits for that duration then loads the last core

### Core Search

```c
const char* findCore(const char* name);
```

Recursively searches directories prefixed with `_` for matching:
- `.rbf` files (FPGA bitstream)
- `.mra` files (MiSTer ROM Archive — arcade definitions)
- `.mgl` files (Multi-Game Launcher scripts)

Search order:
1. Root directory
2. `_` prefixed subdirectories (e.g., `_Console/`, `_Computer/`, `_Arcade/`)
3. Nested subdirectories

### Boot Sequence

```
1. Check if core path was passed via argv (from previous app_restart)
2. Check INI for explicit bootcore setting
3. Check lastcore.dat for previous core
4. Fall back to MENU core
```

---

## 5. Core Lifecycle

### Loading a Core

```
fpga_load_rbf(name, cfg, xml)
    |
    v
do_bridge(0)                  // Disable HPS-FPGA bridges
    |
    v
socfpga_load(rbf_data, size)  // Program FPGA via FPGA Manager
    |
    v
do_bridge(1)                  // Re-enable bridges
    |
    v
app_restart(path, xml, exe)   // Re-exec process with new core
    |
    v
execl(exe, exe, path, xml, NULL)  // Replace process image
```

### Core Initialization (after exec)

```
user_io_init(path, xml)
    |
    v
fpga_core_id()        -> Read core type (validates 0x5CA623 magic)
fpga_get_fio_size()   -> 8-bit or 16-bit file I/O
fpga_get_io_version() -> I/O version (0-3)
    |
    v
UIO_GET_STRING        -> Read config string from core
    |
    v
cfg_parse()           -> Parse MiSTer.ini for core-specific settings
    |
    v
Parse config string options (J, SS, U, C, V, X flags)
    |
    v
Core-specific initialization:
    |-- Minimig: BootInit()
    |-- x86: x86_init()
    |-- Archie: archie_init()
    |-- SharpMZ: sharpmz_init()
    |-- Arcade: arcade_send_rom() (from MRA XML)
    |-- Generic: load boot ROMs, mount VHDs
    |
    v
Load saved status from .CFG file
    |
    v
Release core reset
```

### Core Polling (ongoing)

```
user_io_poll()  -- called every scheduler iteration
    |
    v
user_io_send_buttons()   -- push button/config state
check_status_change()    -- detect core-initiated status updates
SD card emulation        -- sector read/write for up to 16 disks
Keyboard LED polling     -- UIO_GET_KBD_LED every 100ms
Core info polling        -- UIO_INFO_GET
Video mode adjustment    -- UIO_GET_OSDMASK every 500ms
    |
    v
Core-specific polling:
    |-- MegaCD: mcd_poll() -- CD audio/data streaming
    |-- PCE CD: pcecd_poll() -- CD-ROM emulation
    |-- Saturn: saturn_poll() -- CD-ROM emulation
    |-- PSX: psx_poll() -- CD-ROM + save emulation
    |-- NeoGeo CD: neocd_poll() -- CD-ROM emulation
    |-- N64: n64_poll() -- Controller pak, rumble
    |-- Archie: archie_poll() -- FDD emulation
    |-- SharpMZ: sharpmz_poll() -- Tape/keyboard
    |-- x86: x86_poll() -- IDE, share
```

---

## 6. Core Type Detection

All detection functions are lazy-initialized with cached results:

```c
// Pattern (each function follows this structure):
char is_corename() {
    static char cached = 0;
    if (!cached) cached = !strcasecmp(orig_name, "CORENAME") ? 1 : 2;
    return cached == 1;
}
```

| Function | Matches | Core |
|----------|---------|------|
| `is_minimig()` | `"minimig"` | Amiga |
| `is_menu()` | `"MENU"` | MiSTer Menu core |
| `is_x86()` | `"AO486"` | ao486 PC |
| `is_pcxt()` | `"PCXT"` | IBM PC/XT |
| `is_snes()` | `"SNES"` | Super Nintendo |
| `is_sgb()` | `"SGB"` | Super Game Boy |
| `is_neogeo()` | `"neogeo"` | Neo Geo |
| `is_neogeo_cd()` | `is_neogeo() && neocd_is_en()` | Neo Geo CD |
| `is_megacd()` | `"MEGACD"` | Mega CD / Sega CD |
| `is_pce()` | `"TGFX16"` | PC Engine / TurboGrafx |
| `is_archie()` | `"ARCHIE"` | Acorn Archimedes |
| `is_gba()` | `"GBA"` | Game Boy Advance |
| `is_c64()` | `"C64"` | Commodore 64 |
| `is_psx()` | `"PSX"` | PlayStation |
| `is_st()` | `"AtariST"` | Atari ST |
| `is_saturn()` | `"Saturn"` | Sega Saturn |
| `is_n64()` | `"N64"` | Nintendo 64 |
| `is_uneon()` | `"Uneon"` | Uneon |
| `is_sharpmz()` | core_type == `CORE_TYPE_SHARPMZ` | Sharp MZ |
| `is_arcade()` | Set by XML/MRA detection | Arcade cores |

Internal-only:
| `is_cpc()` | `"amstrad"` | Amstrad CPC |
| `is_zx81()` | `"zx81"` | Sinclair ZX81 |
| `is_c128()` | `"C128"` | Commodore 128 |
| `is_electron()` | `"AcornElectron"` | Acorn Electron |

---

## 7. Core-Specific Support Details

### Arcade (MRA Loader)

- Parses MRA XML files to assemble ROM data
- Supports multiple ROM regions, interleaving, byte-swapping
- Uses `buffer.cpp` for ROM data assembly
- Loads ROM data via SPI file transfer protocol
- Supports SDRAM and direct-load addresses

### Minimig (Amiga)

**4 source files:**
- `minimig_config.cpp` — Multi-slot configuration (chipset, CPU, memory, video)
- `minimig_boot.cpp` — Kickstart ROM loading, boot sequence
- `minimig_fdd.cpp` — Floppy disk emulation (ADF format, up to 4 drives)
- `minimig_share.cpp` — Network share / filesystem access

**Uses custom Minimig SPI commands:** `UIO_MM2_*` (0xF0-0xF9)

### x86 (ao486 / PCXT)

- IDE hard disk and floppy emulation
- BIOS ROM loading
- Network share (filesystem access for the emulated PC)
- Uses DMA commands: `UIO_DMA_WRITE` (0x61), `UIO_DMA_READ` (0x62), `UIO_DMA_SDIO` (0x63)

### CD-ROM Cores (MegaCD, PCE CD, Saturn, PSX, NeoGeo CD)

All implement CD-ROM emulation with:
- CUE/BIN and CHD image parsing
- Audio track playback (Red Book CD audio)
- Data sector reading with proper subchannel data
- Seek timing emulation (PCE CD has dedicated `seektime.cpp`)
- Regular polling functions called from `user_io_poll()` and between OSD line transfers

### SNES

- ROM header analysis (LoROM/HiROM detection)
- BSX BIOS support
- SPC file playback
- Special handling for headerless ROMs

### N64

- Controller pak (memory card) emulation
- Rumble feedback (`UIO_GET_RUMBLE`)
- Joystick emulation for C-buttons (`n64_joy_emu.cpp`)

### Commodore 64

- D64/G64/D71/G71/D81 disk image format conversion
- T64 tape image support
- GCR encoding for disk images

### Sharp MZ

- Custom keyboard handling
- Tape format support
- Uses `CORE_TYPE_SHARPMZ` (0xa7) core type

### Atari ST

- TOS ROM loading
- Floppy and hard disk support
- Custom status management (bypasses standard 128-bit status)

---

## 8. Save State Support

Cores can declare save state support via the `SS` parameter in the config string:

```
SS[base_address],[size]
```

### Implementation

```c
int process_ss(const char *rom_name, int enable = 1);
```

- Maps 4 save state slots at `ss_base` via `shmem_map(fpga_mem(ss_base), ...)`
- Each slot is `ss_size` bytes
- A counter at offset 0 of each slot is incremented by the FPGA core when state changes
- HPS polls every 1 second for counter changes
- Changed data is written to disk: `saves/CORENAME/romname_X.ss` (X = slot 0-3)

### Save State Flow

```
FPGA Core:
    1. Writes save state data to SDRAM at ss_base + (slot * ss_size)
    2. Increments counter at offset 0

HPS (user_io_poll):
    1. Maps SDRAM region
    2. Checks if counter changed since last poll
    3. If changed: writes slot data to disk file
    4. Updates cached counter
```

---

## 9. Configuration String Special Parameters

The first config string entry after the core name (index 1) can contain:

| Parameter | Format | Description |
|-----------|--------|-------------|
| `SS` | `SS[base],[size]` | Save state SDRAM base and per-slot size |
| `U` | `U[baud1],[baud2],...` | Supported UART baud rates |
| `C` | `C` | Cheats support enabled |
| `V` | `V,[version]` | Config file version suffix |
| `X` | `X` | OSD disabled (headless mode) |
| `J` | `J[swap],[btn1],[btn2],...` | Joystick button names (comma-separated) |

---

## 10. Direct Memory Loading

For load addresses >= `0x20000000`, file data is written directly to FPGA-accessible SDRAM, bypassing the SPI file transfer protocol entirely:

```c
// In user_io_file_tx():
if (load_addr >= 0x20000000) {
    uint8_t *mem = (uint8_t *)shmem_map(fpga_mem(load_addr), size);
    // Read file data directly into mapped memory
    FileReadAdv(&f, mem, size);
    shmem_unmap(mem, size);
}
```

This is significantly faster than SPI transfer for large files (ROMs, disk images).

---

## 11. Storage System

### Storage Locations

| Priority | Path | Source |
|----------|------|--------|
| 1 | USB (`/media/usb0-5`) | USB drives |
| 2 | Network shares | CIFS/SMB mounts |
| 3 | SD card (`/media/fat`) | Primary storage |
| 4 | `games/` subdirectory | Game data within SD |

### `FindStorage()`

Called at startup:
1. Checks `config/device.bin` for configured storage device
2. Probes USB drives
3. Falls back to SD card root

### Directory Structure

```
/media/fat/
    config/             System configuration
        MiSTer.ini      Main config file
        lastcore.dat    Last loaded core
        *.cfg           Per-core settings
    _Console/           Console cores
    _Computer/          Computer cores
    _Arcade/            Arcade cores
    games/              Game data (ROMs, disk images)
        SNES/           Per-core game directories
        Genesis/
        ...
    saves/              Save data
        SNES/           Per-core save directories
```

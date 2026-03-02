# File I/O System - Complete Reference

## Source Files

| File | Lines | Role |
|------|-------|------|
| `file_io.h` | ~150 | API declarations, constants |
| `file_io.cpp` | 2017 | Implementation with ZIP/directory support |

---

## 1. `fileTYPE` Structure

```c
typedef struct {
    int       fd;           // File descriptor (-1 = closed)
    int       mode;         // Open mode (0=read, 1=read/write, -1=shared memory)
    long      offset;       // Current position in file
    long      size;         // Total file size
    char      name[1024];   // Full file path
    uint32_t  type;         // File type flags

    // ZIP support fields
    void     *zip;          // miniz mz_zip_archive handle (NULL if not a zip)
    int       zip_index;    // File index within zip archive
} fileTYPE;
```

### Mode Values

| Mode | Meaning |
|------|---------|
| 0 | Read-only (`O_RDONLY`) |
| 1 | Read-write (`O_RDWR`) |
| -1 | Shared memory (via `shm_open("/vdsk")`) |

---

## 2. File Operations

### Opening Files

```c
int FileOpen(fileTYPE *file, const char *name, int mode = 0);
```
Opens a file. Returns 1 on success, 0 on failure.

```c
int FileOpenEx(fileTYPE *file, const char *name, int mode, char mute = 0, int use_active = 0);
```
Extended open with additional options:
- `mute`: suppress error logging
- `use_active`: use active storage directory prefix

### Closing Files

```c
void FileClose(fileTYPE *file);
```
Closes the file descriptor (or zip archive) and resets all fields.

### Reading

```c
int FileRead(fileTYPE *file, void *buffer, int size);
```
Reads `size` bytes. For regular files, uses `read()`. For ZIP files, uses miniz extraction. Returns number of bytes read.

```c
int FileReadAdv(fileTYPE *file, void *buffer, int size, int failres = 0);
```
Advanced read with partial-read handling. Retries short reads. Returns `size` on success, `failres` on failure.

### Writing

```c
int FileWrite(fileTYPE *file, const void *buffer, int size);
```
Writes `size` bytes. Returns number of bytes written.

```c
int FileWriteAdv(fileTYPE *file, const void *buffer, int size, int failres = 0);
```
Advanced write with retry logic.

### Seeking

```c
int FileSeek(fileTYPE *file, long offset, int origin);
```
Seeks within the file. For ZIP files, seeks within the extracted data.
- `SEEK_SET`: from beginning
- `SEEK_CUR`: from current position
- `SEEK_END`: from end

### Position and Size

```c
long FileGetSize(fileTYPE *file);
```
Returns file size.

---

## 3. ZIP Support

ZIP archives are transparently supported through the **miniz** library:

### How It Works

1. `FileOpen()` detects `.zip` extension
2. Opens the ZIP archive using `mz_zip_reader_init_file()`
3. If the ZIP contains a single file (or a file matching the expected extension), sets `zip_index`
4. Subsequent `FileRead()` calls extract data from the ZIP entry
5. `FileSeek()` seeks within the extracted data
6. `FileClose()` closes the ZIP archive

### ZIP File Selection

When opening a ZIP, the system:
1. Checks if there's exactly one file matching the expected extension
2. If multiple files, the file browser lets the user select which one
3. The selected file's index is stored in `zip_index`

---

## 4. Directory Scanning

### `ScanDirectory()`

```c
int ScanDirectory(const char* path, int mode, const char* extension, int active = 0);
```

Scans a directory for files matching the given extension filter.

### Scan Mode Flags

```c
#define SCANF_INIT       0    // Initialize scan
#define SCANF_END        1    // Finalize scan
#define SCANF_NEXT       2    // Get next entry
#define SCANF_PREV       3    // Get previous entry
#define SCANF_NEXT_PAGE  4    // Jump forward one page
#define SCANF_PREV_PAGE  5    // Jump backward one page
```

### Scan Option Flags

```c
#define SCANO_DIR        0x01  // Include directories in results
#define SCANO_UMOUNT     0x02  // Allow unmount option
#define SCANO_CORES      0x04  // Scan for core files (.rbf, .mra, .mgl)
#define SCANO_TXT        0x08  // Text file mode
#define SCANO_NEOGEO     0x10  // Neo Geo specific scanning
#define SCANO_NOENTER    0x20  // Don't auto-enter single-entry directories
#define SCANO_CLEAR      0x40  // Clear file list before scan
```

### File List API

```c
typedef struct {
    char name[256];
    char altname[256];
    int  de_type;       // DT_REG (file) or DT_DIR (directory)
    int  sel;           // Selected flag
} flist_entry;

flist_entry* flist_First();     // Get first entry
flist_entry* flist_Last();      // Get last entry
flist_entry* flist_Prev(flist_entry*);   // Previous entry
flist_entry* flist_Next(flist_entry*);   // Next entry
int          flist_nDirEntries();        // Total entry count
flist_entry* flist_SelectedEntry();      // Currently selected entry
```

---

## 5. Configuration File I/O

```c
int FileSaveConfig(const char *filename, void *data, int size);
```
Saves binary data to a config file in the core's home directory.

```c
int FileLoadConfig(const char *filename, void *data, int size);
```
Loads binary data from a config file. Returns 1 on success, 0 on failure.

---

## 6. Utility Functions

```c
int FileLoad(const char *name, void *buffer, int maxsize);
```
Loads an entire file into memory buffer. Returns file size, or 0 on failure.

```c
int FileSave(const char *name, const void *buffer, int size);
```
Saves memory buffer to a file. Returns 1 on success.

```c
int FileExists(const char *name, int active = 0);
```
Checks if a file exists. Returns 1 if it does.

```c
int FileIsDir(const char *name);
```
Checks if a path is a directory.

```c
int PathIsDir(const char *name, int active = 0);
```
Checks if a path exists and is a directory.

```c
const char* GetNameFromPath(const char *path);
```
Extracts filename from a full path.

```c
char* getExtension(const char *name);
```
Returns pointer to the file extension (after the last dot).

---

## 7. Storage Path Functions

```c
void FindStorage();
```
Locates the primary storage device at startup.

```c
const char* getStorageDir(int active);
```
Returns the active storage directory path.

```c
const char* getRootDir();
```
Returns the root directory path (typically `/media/fat`).

```c
const char* HomeDir(const char *suffix);
```
Returns the core's home directory (macro for `user_io_get_core_path(suffix)`).

---

## 8. File Transfer to FPGA

These functions in `user_io.cpp` use the file I/O system to transfer data to/from the FPGA:

### High-Level Transfer

```c
int user_io_file_tx(const char *name, unsigned char index, char opensave,
                    char mute, char composite, uint32_t load_addr);
```

Complete file transfer sequence:
1. Open file via `FileOpen()`
2. Set file index: `FIO_FILE_INDEX` (0x55)
3. Send extension info: `FIO_FILE_INFO` (0x56)
4. Start download: `FIO_FILE_TX` (0x53) with 0xFF
5. Read in 4096-byte chunks, send via `FIO_FILE_TX_DAT` (0x54)
6. End download: `FIO_FILE_TX` (0x53) with 0x00
7. Compute CRC32 of content
8. Show progress bar during transfer

**Special cases:**
- SNES: header analysis (LoROM/HiROM), BSX BIOS, SPC ROM prepend
- GBA: goomba.rom prepend for retro Game Boy
- Electron: UEF file handling
- Composite files: multiple files concatenated
- Direct memory: addresses >= 0x20000000 bypass SPI, use `shmem_map()`

### Disk Image Mount

```c
int user_io_file_mount(const char *name, unsigned char index, char pre, int pre_size);
```

1. Open image file
2. Handle format conversion (D64/G64/T64 for C64, DSK for Apple II, TRD for Spectrum)
3. Send CSD/CID: `UIO_SET_SDCONF` (0x19)
4. Send image size: `UIO_SET_SDINFO` (0x1D)
5. Send mount notification: `UIO_SET_SDSTAT` (0x1C)

---

## 9. Shared Memory File Mode

When `mode = -1`, files are opened via POSIX shared memory:

```c
int fd = shm_open("/vdsk", O_CREAT | O_RDWR, 0666);
ftruncate(fd, size);
```

This creates an in-memory file accessible by other processes, used for virtual disk images in special cases.

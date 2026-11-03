# File System Calls Explained: How a DBMS Talks to the Operating System

### A companion to *Understanding File Systems in Databases*

The **file manager** of a DBMS can't touch the disk directly. Every read, write, flush or resize goes through a **system call**: a controlled request from a user program into the operating-system kernel. This article explains each system call a database uses, what it does inside the OS, and how it applies to our running example.

---

## Table of Contents

1. [What Is a System Call?](#1-what-is-a-system-call)
2. [The Running Example](#2-the-running-example)
3. [Opening and Closing Files: `open`, `close`](#3-opening-and-closing-files)
4. [Basic I/O: `read`, `write`](#4-basic-io-read-and-write)
5. [Moving the File Position: `lseek`](#5-moving-the-file-position-lseek)
6. [Positional I/O: `pread`, `pwrite`](#6-positional-io-pread-and-pwrite)
7. [Scatter/Gather I/O: `readv`, `writev`, `preadv`, `pwritev`](#7-scattergather-io)
8. [Durability: `fsync`, `fdatasync`, `sync_file_range`, `O_SYNC`](#8-durability-fsync-and-friends)
9. [Bypassing the OS Cache: `O_DIRECT`](#9-bypassing-the-os-cache-o_direct)
10. [Growing and Shrinking Files: `ftruncate`, `fallocate`](#10-growing-and-shrinking-files)
11. [File Information: `stat`, `fstat`](#11-file-information-stat-and-fstat)
12. [Giving Hints to the OS: `posix_fadvise`](#12-giving-hints-to-the-os-posix_fadvise)
13. [Memory-Mapped Files: `mmap`, `msync`, `munmap`, `madvise`](#13-memory-mapped-files)
14. [File Locking: `flock`, `fcntl`](#14-file-locking)
15. [Directory and Name Operations: `rename`, `unlink`, `mkdir`](#15-directory-and-name-operations)
16. [Asynchronous I/O: Linux AIO and `io_uring`](#16-asynchronous-io)
17. [Windows Equivalents](#17-windows-equivalents)
18. [Putting It Together: A Full Read-Modify-Write](#18-putting-it-together)
19. [Quick Reference Table](#19-quick-reference-table)

---

## 1. What Is a System Call?

A user program runs in **user mode**, where it can't access hardware or kernel memory. To do I/O it executes a special CPU instruction (`syscall` on x86-64) that switches into **kernel mode**, runs the requested kernel function, and switches back.

```
   USER MODE                         KERNEL MODE
 ┌─────────────────────┐          ┌──────────────────────────────┐
 │ DBMS File Manager   │          │                              │
 │                     │ syscall  │ 1. Validate arguments        │
 │ pread(fd,buf,200,   │ ───────► │ 2. Find the open file (fd)   │
 │       200)          │          │ 3. VFS → file system → cache │
 │                     │          │    → block layer → driver    │
 │   (waits / blocked) │          │ 4. Copy data into buf        │
 │                     │ ◄─────── │ 5. Return result             │
 │ ret = 200 bytes     │  return  │                              │
 └─────────────────────┘          └──────────────────────────────┘
```

**Key facts:**

- Each system call has a **cost** (mode switch, argument checks), so databases try to make fewer, larger calls.
- Most calls return **−1 on error** and set the global variable **`errno`** to explain why (e.g. `ENOENT` = file not found, `EINTR` = interrupted, `ENOSPC` = disk full).
- The C functions you call (`read`, `pread`) are thin wrappers in the C library (glibc) around the actual kernel entry points.

---

## 2. The Running Example

Same as the main article:

| Item | Value |
|------|-------|
| File | `EMPLOYEE.dat` |
| Record size | 40 bytes |
| Block size | 200 bytes (5 records) |
| Block 1 | bytes 0–199 → EmpIDs 101, 105, 110, 115, 120 |
| Block 2 | bytes 200–399 → EmpIDs 125, 130, 135, 140, 145 |

```
 byte 0               200                 400
 ┌────────────────────┬───────────────────┐
 │      BLOCK 1       │      BLOCK 2      │
 └────────────────────┴───────────────────┘
                       ▲          ▲
                       │          └─ EmpID 130 at byte 240
                       └─ block 2 starts at (2 − 1) × 200 = 200
```

> **Note:** 200-byte blocks keep the arithmetic simple. Real databases use 4 KB–16 KB pages, which line up with OS pages and disk sectors (important for `O_DIRECT`, Section 9).

---

## 3. Opening and Closing Files

### 3.1 `open` / `openat`

```c
int open(const char *path, int flags, mode_t mode);
int openat(int dirfd, const char *path, int flags, mode_t mode);
```

**What it does:** Finds the file by name, checks permissions, and returns a **file descriptor (fd)**, a small integer the program uses for all later calls on that file.

```
 open("EMPLOYEE.dat", O_RDWR)  →  fd = 3

 PROCESS FD TABLE          KERNEL OPEN-FILE TABLE         INODE (on disk / cached)
 ┌────┬─────────┐         ┌──────────────────────┐       ┌───────────────────────┐
 │ 0  │ stdin   │         │ flags : O_RDWR       │       │ size   : 400 bytes    │
 │ 1  │ stdout  │         │ offset: 0            │──────►│ owner, permissions    │
 │ 2  │ stderr  │         │ ref count: 1         │       │ block map / extents   │
 │ 3  │ ────────┼────────►└──────────────────────┘       └───────────────────────┘
 └────┴─────────┘
```

Note that the **current file offset** lives in the kernel's open-file entry, not in your program. This matters for `read`/`write` vs `pread`/`pwrite`.

**Important flags:**

| Flag | Meaning | DBMS use |
|------|---------|----------|
| `O_RDONLY` / `O_WRONLY` / `O_RDWR` | Access mode | Data files are opened `O_RDWR` |
| `O_CREAT` | Create the file if missing (needs `mode`, e.g. `0600`) | Creating a new table file |
| `O_EXCL` | With `O_CREAT`: fail if the file already exists | Avoid overwriting an existing table |
| `O_TRUNC` | Cut the file to 0 bytes | Rarely; dangerous on data files |
| `O_APPEND` | Every write goes to the end of the file | Log files |
| `O_SYNC` | Every write waits until data **and** metadata are on disk | Write-ahead log |
| `O_DSYNC` | Every write waits until data (plus metadata needed to read it) is on disk | Write-ahead log, cheaper than `O_SYNC` |
| `O_DIRECT` | Bypass the OS page cache (Linux) | DBMS with its own buffer manager |
| `O_CLOEXEC` | Close fd automatically if the process `exec`s another program | Good hygiene |

**`openat`** opens a file relative to a directory fd. It's safer when the directory might be renamed meanwhile, and it's what modern libraries use internally.

### 3.2 `close`

```c
int close(int fd);
```

Releases the fd. **`close` does not guarantee data is on disk.** Only `fsync` (Section 8) does that.

### Example

```c
int fd = open("EMPLOYEE.dat", O_RDWR | O_CLOEXEC);
if (fd == -1) {
    perror("open");          // e.g. "open: No such file or directory"
    return 1;
}
/* ... use fd ... */
close(fd);
```

---

## 4. Basic I/O: `read` and `write`

```c
ssize_t read (int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

**What they do:** Read or write up to `count` bytes **at the current file offset**, then **advance the offset** by the number of bytes transferred.

```
 Offset before: 0
 read(fd, buf, 200)   → reads bytes 0–199 (Block 1), offset becomes 200
 read(fd, buf, 200)   → reads bytes 200–399 (Block 2), offset becomes 400
 read(fd, buf, 200)   → returns 0  (end of file)
```

**Return values:**

| Return | Meaning |
|--------|---------|
| `> 0` | Number of bytes actually transferred (may be **less** than `count`) |
| `0` | End of file (for `read`) |
| `-1` | Error; check `errno` |

**Short reads/writes:** A call can transfer fewer bytes than asked (end of file, a signal arrived, a network or pipe source). Robust code loops until everything is done:

```c
ssize_t read_full(int fd, char *buf, size_t n) {
    size_t done = 0;
    while (done < n) {
        ssize_t r = read(fd, buf + done, n - done);
        if (r == 0)  break;                 // EOF
        if (r == -1) {
            if (errno == EINTR) continue;   // interrupted, retry
            return -1;
        }
        done += r;
    }
    return done;
}
```

**Use in a DBMS:** Plain `read` is perfect for a **full sequential scan**: just keep reading block after block.

**Problem:** To read block 2 directly you must first move the offset (`lseek`), and the shared offset causes trouble with multiple threads (next sections).

---

## 5. Moving the File Position: `lseek`

```c
off_t lseek(int fd, off_t offset, int whence);
```

| `whence` | New offset = |
|----------|--------------|
| `SEEK_SET` | `offset` |
| `SEEK_CUR` | current + `offset` |
| `SEEK_END` | file size + `offset` |

**Reading block 2 with `lseek` + `read`:**

```c
lseek(fd, 200, SEEK_SET);   // move to the start of block 2
read(fd, buf, 200);         // read block 2
```

**Other uses:**

- `lseek(fd, 0, SEEK_END)` returns the file size (to find where to append a new block).
- `SEEK_DATA` / `SEEK_HOLE` (Linux) find data regions in sparse files.

**The race condition problem.** The offset is shared by all threads using the same fd:

```
 Thread A (wants block 2)          Thread B (wants block 1)
 lseek(fd, 200, SEEK_SET)
                                   lseek(fd, 0, SEEK_SET)      ← overwrites A's offset!
 read(fd, buf, 200)  → gets BLOCK 1 ✗ wrong data
                                   read(fd, buf, 200) → gets BLOCK 2 ✗
```

Two calls (seek, then read) are **not atomic**. That's why databases use `pread`.

---

## 6. Positional I/O: `pread` and `pwrite`

```c
ssize_t pread (int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

**What they do:** Read or write `count` bytes **at the given offset**, in **one atomic call**, **without changing** the file's current offset.

```
 pread(fd, buf, 200, 200)

 byte 0               200                 400
 ┌────────────────────┬───────────────────┐
 │      BLOCK 1       │█████ BLOCK 2 █████│  → 200 bytes copied into buf
 └────────────────────┴───────────────────┘
 file offset is untouched (still whatever it was)
```

### Why databases prefer `pread`/`pwrite`

| Benefit | Explanation |
|---------|-------------|
| **One system call instead of two** | No separate `lseek` → less overhead |
| **Thread-safe** | Each thread passes its own offset; no shared state to corrupt |
| **Natural for block addressing** | Offset = (block number − 1) × block size, computed directly |

The same race as before is now safe:

```
 Thread A: pread(fd, bufA, 200, 200)   → always Block 2 ✓
 Thread B: pread(fd, bufB, 200,   0)   → always Block 1 ✓   (can run concurrently)
```

### Example: read the block containing EmpID 130 and extract the record

```c
#define BLOCK_SIZE  200
#define RECORD_SIZE 40

char block[BLOCK_SIZE];
int  block_no = 2;                                    // from the index
off_t offset  = (off_t)(block_no - 1) * BLOCK_SIZE;   // = 200

ssize_t n = pread(fd, block, BLOCK_SIZE, offset);
if (n != BLOCK_SIZE) { /* handle error or short read */ }

int slot = 1;                                         // 2nd record in the block
char *record = block + slot * RECORD_SIZE;            // byte 40 inside block

int32_t empid;  memcpy(&empid,  record + 2,  4);      // offset 2  → 130
int32_t salary; memcpy(&salary, record + 36, 4);      // offset 36 → 80000
```

### Example: update Gauri's salary in place

```c
int32_t new_salary = 85000;
memcpy(record + 36, &new_salary, 4);                  // change in memory
pwrite(fd, block, BLOCK_SIZE, offset);                // write whole block back
```

> The DBMS writes the **whole block**, not just 4 bytes, because the block is the unit of I/O and of the buffer manager.

**Important:** after `pwrite` returns, the data is usually only in the **OS page cache**, not yet on disk. A power failure could lose it. Section 8 fixes that.

---

## 7. Scatter/Gather I/O

```c
ssize_t readv  (int fd, const struct iovec *iov, int iovcnt);
ssize_t writev (int fd, const struct iovec *iov, int iovcnt);
ssize_t preadv (int fd, const struct iovec *iov, int iovcnt, off_t offset);
ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt, off_t offset);

struct iovec { void *iov_base; size_t iov_len; };
```

**What they do:** Transfer data between **one contiguous region of the file** and **several separate memory buffers**, in one system call.

```
 preadv(fd, iov, 2, 0)         FILE: bytes 0..399 (contiguous)
                               ┌───────────────┬───────────────┐
                               │   BLOCK 1     │   BLOCK 2     │
                               └───────┬───────┴───────┬───────┘
                         scatter       │               │
                                       ▼               ▼
 MEMORY (buffer pool, not adjacent) [frame 7]       [frame 42]
```

**DBMS use:**

- Read several consecutive disk blocks into **free buffer-pool frames that are not next to each other** in memory, with one call.
- Write a log record made of a header buffer and a payload buffer without first copying them together (`writev`).

---

## 8. Durability: `fsync` and Friends

### 8.1 The problem: writes are cached

```
 pwrite() returns here ──┐
                         ▼
 DBMS buffer ──► OS PAGE CACHE (RAM) ──► disk drive cache ──► platter / flash
                 "dirty page"            (volatile too!)       PERSISTENT
                 lost on power failure
```

The OS writes dirty pages to disk **whenever it chooses** (often seconds later). A database that promised "your transaction is committed" must be sure the data survives a crash. That's the **D** in ACID.

### 8.2 `fsync`

```c
int fsync(int fd);
```

Blocks until **all modified data and metadata** of the file (size, modification time, block map) are transferred to the storage device, and asks the device to flush its own cache.

### 8.3 `fdatasync`

```c
int fdatasync(int fd);
```

Like `fsync`, but skips metadata that isn't needed to read the data back (e.g. last-modified time). **Faster**, and databases use it for log files whose size doesn't change (pre-allocated).

| | `fsync` | `fdatasync` |
|---|---------|-------------|
| File data | ✓ | ✓ |
| File size change | ✓ | ✓ (needed to read data) |
| Timestamps, permissions | ✓ | ✗ |
| Speed | Slower | Faster |

### 8.4 `O_SYNC` / `O_DSYNC` (flags on `open`)

Make **every write** behave as if followed by `fsync` (`O_SYNC`) or `fdatasync` (`O_DSYNC`). Convenient for a write-ahead log where each write must be durable anyway.

### 8.5 `sync_file_range` (Linux)

```c
int sync_file_range(int fd, off64_t offset, off64_t nbytes, unsigned int flags);
```

Starts or waits for write-back of **just a byte range**. Databases use it to start flushing early and spread the I/O out during checkpoints. **It is not a durability guarantee**: it doesn't flush metadata or the drive cache. A real `fsync` is still required at the end.

### 8.6 How a DBMS commits a transaction (write-ahead logging)

```
 UPDATE EMPLOYEE SET Salary = 85000 WHERE EmpID = 130;  COMMIT;

 1. Change block 2 in the DBMS buffer (in memory only)
 2. pwrite(log_fd, log_record, ...)       ← "EmpID 130: 80000 → 85000"
 3. fdatasync(log_fd)                     ← log is now safely on disk
 4. Tell the user: COMMIT OK
 5. Later (checkpoint): pwrite(data_fd, block2, 200, 200); fsync(data_fd)
```

Only the small log write is forced at commit time; the data block can be written lazily. If the system crashes before step 5, recovery replays the log.

### 8.7 A subtle trap: new files need the directory synced too

When you **create** or **rename** a file, the file's name lives in the **directory**. To make the new name durable:

```c
int fd = open("EMPLOYEE.dat", O_RDWR | O_CREAT | O_EXCL, 0600);
pwrite(fd, data, len, 0);
fsync(fd);                                   // file contents durable

int dfd = open(".", O_RDONLY | O_DIRECTORY);
fsync(dfd);                                  // directory entry durable
close(dfd);
```

### 8.8 Error handling matters

If `fsync` returns `-1` with `EIO`, some data may already be **lost**, and on Linux, retrying `fsync` may wrongly report success because the failed pages were marked clean. PostgreSQL (since 2018) treats an `fsync` failure as fatal and restarts into crash recovery for exactly this reason.

---

## 9. Bypassing the OS Cache: `O_DIRECT`

```c
int fd = open("EMPLOYEE.dat", O_RDWR | O_DIRECT);
```

### 9.1 Why

Without `O_DIRECT`, the same block is cached **twice**:

```
 WITHOUT O_DIRECT                         WITH O_DIRECT
 ┌──────────────────┐                     ┌──────────────────┐
 │ DBMS buffer pool │ block 2             │ DBMS buffer pool │ block 2
 └────────▲─────────┘                     └────────▲─────────┘
          │ copy                                   │ DMA straight
 ┌────────┴─────────┐                              │ from device
 │ OS page cache    │ block 2 (again!)             │
 └────────▲─────────┘                              │
          │                                        │
       [ disk ]                                 [ disk ]
```

- Avoids wasting RAM on two copies
- Avoids an extra memory copy
- Lets the DBMS buffer manager, which knows the query workload, decide what to cache

Used by Oracle, MySQL InnoDB (`innodb_flush_method=O_DIRECT`), and others. PostgreSQL traditionally relies on the OS cache instead.

### 9.2 The alignment rule

With `O_DIRECT`, the **buffer address**, **file offset**, and **length** must all be multiples of the device's logical block size (usually 512 B or 4096 B). Otherwise the call fails with `EINVAL`.

```
 pread(fd, buf, 200, 200)   ✗ EINVAL: 200 is not a multiple of 512/4096
 pread(fd, buf, 4096, 4096) ✓ (with buf aligned to 4096)
```

This is one reason real databases use page sizes like 4 KB, 8 KB, or 16 KB rather than 200 bytes.

```c
void *buf;
posix_memalign(&buf, 4096, 8192);   // allocate an aligned 8 KB buffer
pread(fd, buf, 8192, 8192);         // read page 2 of an 8 KB-page file
```

### 9.3 `O_DIRECT` is not durability

`O_DIRECT` skips the page cache, but data may still sit in the drive's cache and metadata may not be updated. **You still need `fsync`/`fdatasync`** (or `O_DSYNC`) for durability.

---

## 10. Growing and Shrinking Files

### 10.1 `ftruncate`

```c
int ftruncate(int fd, off_t length);
```

Sets the file size to exactly `length` bytes.

- **Shrink:** data beyond `length` is discarded.
- **Grow:** the new region reads as zeros, but usually no disk space is allocated yet (a *sparse file*, with a "hole").

```
 ftruncate(fd, 600)    // EMPLOYEE.dat grows from 400 to 600 bytes
 ┌─────────┬─────────┬ ─ ─ ─ ─ ─ ┐
 │ BLOCK 1 │ BLOCK 2 │  (hole)   │  ← block 3 reads as zeros, not yet on disk
 └─────────┴─────────┴ ─ ─ ─ ─ ─ ┘
```

**DBMS use:** removing empty pages at the end of a table after `VACUUM`/reorganization; shrinking log files.

### 10.2 `fallocate` / `posix_fallocate`

```c
int fallocate(int fd, int mode, off_t offset, off_t len);        // Linux
int posix_fallocate(int fd, off_t offset, off_t len);            // portable
```

**Actually reserves disk space** for the range, so later writes:

- can't fail with "disk full" (`ENOSPC`)
- are more likely to be **physically contiguous** (good for sequential scans)
- don't need metadata updates, so `fdatasync` is cheaper

```
 fallocate(fd, 0, 400, 1000)  // reserve space for blocks 3–7 in advance

 ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
 │ BLOCK 1 │ BLOCK 2 │ resvd 3 │ resvd 4 │ resvd 5 │ resvd 6 │ resvd 7 │
 └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Linux `fallocate` modes also include `FALLOC_FL_PUNCH_HOLE` (free space in the middle of a file) and `FALLOC_FL_KEEP_SIZE` (reserve without changing the reported size).

**DBMS use:** pre-allocating log segments and table extents.

---

## 11. File Information: `stat` and `fstat`

```c
int stat (const char *path, struct stat *st);
int fstat(int fd, struct stat *st);
```

Fills a `struct stat` with information from the inode:

| Field | Meaning | Example for EMPLOYEE.dat |
|-------|---------|--------------------------|
| `st_size` | Size in bytes | 400 |
| `st_blocks` | 512-byte units actually allocated | 8 (one 4 KB fs block) |
| `st_blksize` | Preferred I/O size | 4096 |
| `st_mode` | Type + permissions | regular file, 0600 |
| `st_mtime` | Last modification time | ... |
| `st_ino` | Inode number | 1835012 |

**DBMS use:**

```c
struct stat st;
fstat(fd, &st);
long num_blocks = st.st_size / BLOCK_SIZE;   // 400 / 200 = 2 blocks
```

The file manager uses this to know how many blocks a table has, and the `st_blksize` hint to choose I/O sizes.

---

## 12. Giving Hints to the OS: `posix_fadvise`

```c
int posix_fadvise(int fd, off_t offset, off_t len, int advice);
```

Tells the kernel how the program **intends** to access a file range, so it can tune read-ahead and caching. It's only a hint; correctness never depends on it.

| Advice | Meaning | DBMS example |
|--------|---------|--------------|
| `POSIX_FADV_SEQUENTIAL` | Will read in order → read ahead aggressively | Full table scan of EMPLOYEE |
| `POSIX_FADV_RANDOM` | Random access → disable read-ahead | Index lookups (B+ tree) |
| `POSIX_FADV_WILLNEED` | Will need this range soon → start loading now | Prefetch the next leaf pages |
| `POSIX_FADV_DONTNEED` | Done with this range → drop it from cache | After a one-time bulk load or backup |
| `POSIX_FADV_NOREUSE` | Access once only | Streaming reads |

```c
// About to scan the whole table:
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);   // len 0 = to end of file

// About to fetch block 2 via the index:
posix_fadvise(fd, 200, 200, POSIX_FADV_WILLNEED);
```

PostgreSQL uses `WILLNEED` to prefetch heap pages during bitmap index scans.

---

## 13. Memory-Mapped Files

### 13.1 `mmap` / `munmap`

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int   munmap(void *addr, size_t length);
```

Maps a file range into the process's address space. Afterwards the program reads and writes the file **like an array in memory**. The OS loads pages on first access (a *page fault*) and writes dirty pages back later.

```
 PROCESS VIRTUAL MEMORY                        EMPLOYEE.dat
 ┌──────────────────────────┐
 │ ...                      │
 │ p ──► [ BLOCK 1 | BLOCK 2 ] ◄── mapped ───► bytes 0–399
 │ ...                      │      (via the page cache)
 └──────────────────────────┘
```

```c
char *p = mmap(NULL, 400, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

int32_t salary;
memcpy(&salary, p + 240 + 36, 4);        // EmpID 130's salary: 80000, no read() call

int32_t new_salary = 85000;
memcpy(p + 240 + 36, &new_salary, 4);    // modify the file through memory
```

> The `offset` argument of `mmap` must be a multiple of the OS page size (usually 4096).

### 13.2 `msync`

```c
int msync(void *addr, size_t length, int flags);   // MS_SYNC or MS_ASYNC
```

The `mmap` equivalent of `fsync`: forces modified mapped pages to disk.

### 13.3 `madvise`

```c
int madvise(void *addr, size_t length, int advice);  // MADV_SEQUENTIAL, MADV_RANDOM, MADV_WILLNEED ...
```

Like `posix_fadvise`, but for a mapped memory region.

### 13.4 Should a DBMS use `mmap`?

| Advantages | Disadvantages |
|------------|---------------|
| No system call per access; simple pointer code | The **OS decides** when dirty pages are written. That can break write-ahead-log ordering unless carefully managed |
| No double caching | I/O errors show up as a crash signal (`SIGBUS`) instead of an error code |
| Zero-copy reads | Page faults stall the thread unpredictably; hard to do async I/O |
| | Harder to control eviction policy |

Some embedded/key-value stores (LMDB, older MongoDB MMAPv1) are built on `mmap`. Most large relational DBMSs (PostgreSQL, MySQL InnoDB, Oracle) use `pread`/`pwrite` with their own buffer manager for control and safety.

---

## 14. File Locking

Prevents two processes from corrupting the same file, for example two copies of the database server starting on the same data directory.

### 14.1 `flock` (whole-file lock)

```c
int flock(int fd, int operation);   // LOCK_SH, LOCK_EX, LOCK_UN, | LOCK_NB
```

```c
int lockfd = open("data/LOCK", O_RDWR | O_CREAT, 0600);
if (flock(lockfd, LOCK_EX | LOCK_NB) == -1) {
    fprintf(stderr, "Another server is already using this data directory\n");
    exit(1);
}
```

### 14.2 `fcntl` record locks (byte-range lock)

```c
struct flock fl = {
    .l_type   = F_WRLCK,     // F_RDLCK (shared), F_WRLCK (exclusive), F_UNLCK
    .l_whence = SEEK_SET,
    .l_start  = 200,         // lock block 2 only
    .l_len    = 200
};
fcntl(fd, F_SETLKW, &fl);    // F_SETLK = try, F_SETLKW = wait
```

```
 EMPLOYEE.dat
 ┌────────────────────┬────────────────────┐
 │ BLOCK 1 (free)     │ BLOCK 2 (LOCKED by │
 │                    │ process 4521)      │
 └────────────────────┴────────────────────┘
```

> **Note:** these are **process-level** locks managed by the OS. Row- and page-level locking for transactions is done by the DBMS's own **lock manager** in memory. It's far faster and understands transactions, which the OS doesn't. OS locks are mainly used to stop multiple server instances from sharing files.

---

## 15. Directory and Name Operations

| Call | Signature | What it does | DBMS use |
|------|-----------|--------------|----------|
| `mkdir` | `int mkdir(const char *path, mode_t mode)` | Create a directory | Create a database / tablespace directory |
| `rmdir` | `int rmdir(const char *path)` | Remove an empty directory | Drop a database |
| `unlink` | `int unlink(const char *path)` | Remove a file name; space is freed when no fd is open on it | `DROP TABLE` → delete `EMPLOYEE.dat` |
| `rename` | `int rename(const char *old, const char *new)` | Atomically change a file's name, replacing `new` if it exists | Safely replace a file |
| `opendir`/`readdir` | (library over `getdents64`) | List a directory | Find all table/segment files at startup |

### The atomic-rename pattern

`rename` within the same file system is **atomic**: other processes see either the old file or the new one, never a half-written mix. Databases use it to replace files safely (e.g. after reorganizing a table, or updating a control/config file):

```
 1. Write the reorganized table to   EMPLOYEE.dat.tmp
 2. fsync(EMPLOYEE.dat.tmp)
 3. rename("EMPLOYEE.dat.tmp", "EMPLOYEE.dat")   ← atomic switch
 4. fsync(directory)                               ← make the rename durable
```

If the system crashes before step 3, the old file is intact. After step 4, the new file is permanent.

---

## 16. Asynchronous I/O

### 16.1 The problem with blocking calls

`pread` **blocks** the calling thread until the data arrives (~0.1 ms on an SSD, ~10 ms on a hard disk). A database wants to keep **many** I/Os in flight at once, for example prefetching 32 index pages, without one thread per I/O.

```
 SYNCHRONOUS (pread)                     ASYNCHRONOUS
 thread: [req1]─wait─[req2]─wait─[req3]  thread: [req1][req2][req3] ... do other work
                                                    ▼     ▼     ▼
                                         device:   (all three in progress)
                                         thread: ... [collect 3 completions]
```

### 16.2 Linux native AIO

```c
io_setup(nr_events, &ctx);              // create an AIO context
io_submit(ctx, n, iocbs);               // submit n read/write requests
io_getevents(ctx, min, max, events, t); // wait for/collect completions
io_destroy(ctx);
```

Works well **only with `O_DIRECT`**; with buffered files it often silently becomes synchronous. MySQL InnoDB uses it (`innodb_use_native_aio`).

### 16.3 `io_uring` (Linux 5.1+)

A newer, faster interface built on two **ring buffers shared between the program and the kernel**:

```
       USER SPACE                               KERNEL
 ┌──────────────────────┐   submission queue   ┌──────────────────┐
 │ put: READ fd=3       │ ───── (SQ) ────────► │ performs I/O     │
 │      off=200 len=200 │                      │ (buffered or     │
 │ put: READ fd=3       │                      │  O_DIRECT)       │
 │      off=0   len=200 │                      │                  │
 │                      │ ◄──── (CQ) ───────── │ posts results    │
 │ get: result=200 ✓    │   completion queue   │                  │
 └──────────────────────┘                      └──────────────────┘
```

- Underlying system calls: `io_uring_setup`, `io_uring_enter`, `io_uring_register` (usually used via the **liburing** library).
- Many requests can be submitted and reaped with **one** system call, or even zero in polling mode.
- Works for buffered and direct I/O, and also for `fsync`, network sockets, etc.
- PostgreSQL 18 added `io_uring` as an option for its new asynchronous I/O subsystem.

```c
// liburing sketch: read block 2 asynchronously
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, block, 200, 200);   // fd, buf, len, offset
io_uring_submit(&ring);

/* ... do other work ... */

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);                 // cqe->res == 200 on success
io_uring_cqe_seen(&ring, cqe);
```

---

## 17. Windows Equivalents

The concepts are identical; the names differ (Win32 API, which calls NT kernel functions underneath).

| POSIX / Linux | Windows (Win32) | Notes |
|---------------|-----------------|-------|
| `open` | `CreateFileW` | Returns a `HANDLE` instead of an int fd |
| `close` | `CloseHandle` | |
| `read` / `write` | `ReadFile` / `WriteFile` | |
| `pread` / `pwrite` | `ReadFile` / `WriteFile` with an `OVERLAPPED` struct holding the offset | |
| `lseek` | `SetFilePointerEx` | |
| `fsync` | `FlushFileBuffers` | |
| `O_DIRECT` | `FILE_FLAG_NO_BUFFERING` | Same alignment rules (sector size) |
| `O_SYNC` / `O_DSYNC` | `FILE_FLAG_WRITE_THROUGH` | |
| `ftruncate` | `SetFileInformationByHandle` (end of file) / `SetEndOfFile` | |
| `fstat` | `GetFileInformationByHandleEx`, `GetFileSizeEx` | |
| `mmap` | `CreateFileMappingW` + `MapViewOfFile` | |
| `flock` / `fcntl` locks | `LockFileEx` / `UnlockFileEx` | |
| `rename` | `MoveFileExW` (`MOVEFILE_REPLACE_EXISTING`) | |
| `unlink` | `DeleteFileW` | |
| Async I/O / `io_uring` | Overlapped I/O + **I/O Completion Ports** (`CreateIoCompletionPort`, `GetQueuedCompletionStatus`); newer **IoRing** API | SQL Server uses completion ports |

---

## 18. Putting It Together

A complete flow: **`UPDATE EMPLOYEE SET Salary = 85000 WHERE EmpID = 130; COMMIT;`**

```
 STEP  DBMS ACTION                                   SYSTEM CALL(S)
 ────  ────────────────────────────────────────────  ───────────────────────────────────
  0    Server start: lock data dir, open files       flock(LOCK), open(EMPLOYEE.dat),
                                                     open(wal.log, O_DSYNC), fstat()
  1    Index says EmpID 130 → block 2                (index page maybe already cached)
  2    Buffer miss → read block 2                    pread(fd, buf, 200, 200)
  3    Change salary in buffer (memory only)         — none —
  4    Write log record                              pwrite(wal_fd, rec, len, log_off)
       (O_DSYNC makes it durable on return,          [or pwrite + fdatasync(wal_fd)]
        or call fdatasync explicitly)
  5    Reply "COMMIT OK" to client                   — (network send) —
  6    Later, checkpoint: write dirty block          pwrite(fd, buf, 200, 200)
  7    Make data file durable                        fsync(fd)
  8    Log space before checkpoint can be reused     (ftruncate / reuse log segment)
  9    Shutdown                                      close(fd), close(wal_fd)
```

### Full C sketch (simplified, no WAL)

```c
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdint.h>
#include <stdio.h>

#define BLOCK_SIZE  200
#define RECORD_SIZE 40

int main(void) {
    int fd = open("EMPLOYEE.dat", O_RDWR | O_CLOEXEC);
    if (fd == -1) { perror("open"); return 1; }

    char  block[BLOCK_SIZE];
    off_t off = (off_t)(2 - 1) * BLOCK_SIZE;               // block 2 → offset 200

    if (pread(fd, block, BLOCK_SIZE, off) != BLOCK_SIZE) { perror("pread"); return 1; }

    for (int slot = 0; slot < BLOCK_SIZE / RECORD_SIZE; slot++) {
        char   *rec = block + slot * RECORD_SIZE;
        int32_t id;  memcpy(&id, rec + 2, 4);
        if (id == 130) {
            int32_t sal = 85000;
            memcpy(rec + 36, &sal, 4);                     // update Salary field
            break;
        }
    }

    if (pwrite(fd, block, BLOCK_SIZE, off) != BLOCK_SIZE) { perror("pwrite"); return 1; }
    if (fsync(fd) == -1) { perror("fsync"); return 1; }    // durable on disk

    close(fd);
    return 0;
}
```

---

## 19. Quick Reference Table

| System call | Purpose | Changes file offset? | Typical DBMS use |
|-------------|---------|----------------------|------------------|
| `open` / `openat` | Get a file descriptor | — | Open table, index, log files |
| `close` | Release the fd | — | Shutdown, drop table |
| `read` / `write` | I/O at current offset | Yes | Sequential scans, log appends |
| `lseek` | Set current offset | Yes | Find file end; legacy random I/O |
| **`pread` / `pwrite`** | I/O at an explicit offset, atomic | **No** | **Random block access (main workhorse)** |
| `readv` / `writev` / `preadv` / `pwritev` | Many buffers, one call | v: yes / pv: no | Multi-block reads into the buffer pool |
| **`fsync`** | Flush data + metadata to disk | — | **Checkpoints, durability** |
| `fdatasync` | Flush data (+ needed metadata) | — | Commit: flush the log |
| `sync_file_range` | Start/wait write-back of a range | — | Smooth out checkpoint I/O |
| `ftruncate` | Set file size | — | Shrink table after vacuum |
| `fallocate` | Reserve disk space | — | Pre-allocate log/data extents |
| `fstat` | File size and info | — | Count blocks in a table |
| `posix_fadvise` | Access-pattern hint | — | Scan vs index lookup, prefetch |
| `mmap` / `msync` / `munmap` | Map file into memory | — | Some embedded / KV stores |
| `flock` / `fcntl` | OS file / range locks | — | Single-instance data directory lock |
| `rename` | Atomic file replace | — | Safe file swap after reorganization |
| `unlink` | Delete a file | — | `DROP TABLE` |
| `mkdir` / `rmdir` | Directories | — | `CREATE` / `DROP DATABASE` |
| `io_submit` / `io_uring_enter` | Asynchronous batched I/O | — | High-throughput prefetch and writes |

---

*End of article.*

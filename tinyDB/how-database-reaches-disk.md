# How Does a Database Reach the Physical Disk?

### Page numbers, the operating system, and who knows where your data really is - From B-tree page numbers to physical bits on an HDD or SSD

---

## The question

A database saves its records on a disk. A B-tree page says things like "the child you want is page 16". But a disk has physical locations: sectors on a platter, or cells in a flash chip. And the operating system hides those details from programs.

So:

1. How does a database point to a physical location on the disk? (How does the database know the exact page number of a page?)
2. Does the operating system give the database those locations? Is the page number the physical location on the disk?
3.  If not, who provides the physical location, and how is the page number mapped to it?
3. Are those locations (page number) persistent? Are they the same after a power-off?
4. After a power-off, is the location the same?
5. Does the database engine know where the root page is?
6. If it is, who saved those locations: the database, the OS, or the SSD/HDD itself?

This article answers each question using a small Java storage engine, a real trace of what that engine asks the operating system to do, and the operating system's own map of where the file sits on disk.

---

## Short answer

- **The database never knows a physical location.** It only knows *"file X, page N"*, and turns that into *"file X, byte offset N × 8,192"*.
- **The OS does not give locations to the database.** The database hands the OS a file and a byte offset. The OS works out where that is on the device, does the read or write, and returns only the bytes. The physical address never comes back to the database.
- **Every layer saves its own mapping on disk**, so all of them survive a power-off. The one exception is the SSD's internal flash location, which moves on every write, but the SSD tracks that itself.
- **The database does know where the root is**, as a page number stored in page 0 of the index file (the metapage). It finds the file itself through its system catalog.

---

## 1. The layers between a database and the disk

```text
 ┌──────────────────────────────────────────────────────────────────────┐
 │  DATABASE ENGINE                                                     │
 │  knows:  file name + page number         e.g. users_pkey.idx, page 18│
 │  saves:  page numbers inside its own pages (metapage, parents)       │
 └───────────────────────────────────┬──────────────────────────────────┘
                                     │  pread(fd, buf, 8192, 18 × 8192)
                                     │  "read 8 KB at offset 147,456 of this file"
 ════════════════════════════════════╪══════ system call boundary ══════
                                     ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  OPERATING SYSTEM — FILESYSTEM (ext4, XFS, NTFS, APFS)               │
 │  knows:  file offset → disk block        e.g. block 36 → 3,153,252   │
 │  saves:  the map in the file's inode / extent tree, on the disk      │
 └───────────────────────────────────┬──────────────────────────────────┘
                                     │  "read sectors starting at LBA 25,226,016"
                                     ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  STORAGE DEVICE FIRMWARE                                             │
 │  knows:  LBA → physical location                                     │
 │          HDD: platter / track / sector                               │
 │          SSD: flash chip / erase block / flash page                  │
 │  saves:  its own tables, inside the drive                            │
 └──────────────────────────────────────────────────────────────────────┘
```

Each layer only understands the address format of the layer directly above it, and only translates it one step down:

| Layer          | Address it receives        | Address it produces      | Who decides the mapping |
|----------------|----------------------------|--------------------------|-------------------------|
| Database       | page number                | file + byte offset       | Database (arithmetic)   |
| Filesystem     | file + byte offset         | LBA (logical block address) | Operating system     |
| Drive firmware | LBA                        | physical location        | HDD or SSD controller   |

An **LBA** is just a block number on the device, counting from 0. Every drive presents itself to the OS as one long array of numbered blocks.

---

## 2. The database side: a working mini storage engine

The Java program below is a tiny storage engine. It stores a B-tree index in a file called `users_pkey.idx`, using 8 KB pages, and it never refers to a disk location of any kind. Everything it does is in terms of page numbers and file offsets.

### 2.1 The three rules that produce page numbers

**Rule 1: a page number is a position in the file.**

```java
static long offsetOf(int pageNo) {
    return (long) pageNo * PAGE_SIZE;   // page 18 -> 147,456
}
```

There is no table and no lookup. Page 18 is simply the 19th 8 KB chunk of the file.

**Rule 2: a new page gets the next free number.**

```java
int allocatePage() throws IOException {
    int newPageNo = pageCount();                           // fileSize ÷ 8,192
    writePage(newPageNo, ByteBuffer.allocate(PAGE_SIZE));  // file grows by 8 KB
    return newPageNo;
}
```

If the file is 19 pages long, the new page becomes page 19. The database chooses the number, not the OS. The OS is only asked to make the file 8 KB longer, and it decides on its own where on the device those new bytes go.

**Rule 3: page numbers are saved inside other pages.**

```java
// Inside each parent (internal) page: the child's first key and page number
buf.putLong(c.firstKey()).putInt(c.pageNo());

// Inside page 0 (the metapage): the root's page number
metaBuf.putInt(MAGIC).putInt(rootPageNo).putInt(levels);
```

These numbers are written to the file like any other data, so they are as persistent as the records themselves.

### 2.2 How the root is found

The root is not "always page 1". In this demo it ends up as **page 18**, because the tree is built bottom-up and the root is the last page created. The only fixed location is page 0, and page 0 says where the root is:

```text
users_pkey.idx

 page 0      pages 1 … 13         pages 14 … 17        page 18
┌──────────┬──────────────────┬───────────────────┬──────────┐
│ METAPAGE │   13 leaf pages  │ 4 internal pages  │   ROOT   │
│ root=18  │   (level 0)      │ (level 1)         │ (level 2)│
└────┬─────┴──────────────────┴───────────────────┴────▲─────┘
     └─────────────────────────────────────────────────┘
          "root = page 18" is stored in page 0
```

When a real database's root splits, it creates a new root page somewhere at the end of the file and updates page 0 to point at it. This is how PostgreSQL B-tree indexes work: block 0 of every index file is a metapage holding the root's block number.

### 2.3 How the file is found

Something must also tell the database which file holds the index. In this demo, the file name is hard-coded. In a real database, a **system catalog** records it:

```text
PostgreSQL:  pg_class row for "users_pkey"  →  relfilenode 24576  →  file base/16384/24576
```

The catalog is itself stored in files with well-known fixed names, which is how the database bootstraps itself at startup.

---

## 3. The full code

Save it as `PagerDemo.java` and run it with `java PagerDemo.java` (Java 16 or newer). To keep the output readable it uses a fanout of 4 entries per page instead of a realistic ~407.

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.ArrayList;
import java.util.List;

/**
 * Shows how a database arrives at page numbers:
 *
 *   1. A page number is a position inside a file:  offset = pageNo × PAGE_SIZE.
 *   2. A new page gets the next free number:       pageNo = fileSize ÷ PAGE_SIZE.
 *   3. Page 0 is a metapage that stores the root's page number.
 *   4. Parent pages store their children's page numbers.
 *   5. All of it is written to the file, so it survives a restart.
 *
 * Run with:  java PagerDemo.java
 */
public class PagerDemo {

    // ------------------------------------------------------------------
    // Layer 1: the Pager. It only knows "page number <-> bytes in a file".
    // ------------------------------------------------------------------
    static final int PAGE_SIZE = 8192;

    static class Pager implements AutoCloseable {
        private final FileChannel file;

        Pager(Path path) throws IOException {
            file = FileChannel.open(path,
                    StandardOpenOption.CREATE,
                    StandardOpenOption.READ,
                    StandardOpenOption.WRITE);
        }

        /** Page number -> byte offset. Pure arithmetic, no lookup table. */
        static long offsetOf(int pageNo) {
            return (long) pageNo * PAGE_SIZE;
        }

        /** Number of pages currently in the file. */
        int pageCount() throws IOException {
            return (int) (file.size() / PAGE_SIZE);
        }

        /** A new page always gets the next number: it is appended at the end of the file. */
        int allocatePage() throws IOException {
            int newPageNo = pageCount();
            writePage(newPageNo, ByteBuffer.allocate(PAGE_SIZE)); // extend the file by 8 KB
            return newPageNo;
        }

        ByteBuffer readPage(int pageNo) throws IOException {
            ByteBuffer buf = ByteBuffer.allocate(PAGE_SIZE);
            file.read(buf, offsetOf(pageNo));
            buf.flip();
            return buf;
        }

        void writePage(int pageNo, ByteBuffer buf) throws IOException {
            buf.rewind();
            file.write(buf, offsetOf(pageNo));
        }

        @Override
        public void close() throws IOException {
            file.force(true); // ask the OS to flush to the drive (fsync)
            file.close();
        }
    }

    // ------------------------------------------------------------------
    // Layer 2: page formats.
    // ------------------------------------------------------------------

    /** Page 0: the metapage. Layout: [magic:int][rootPageNo:int][levels:int] */
    static final int META_PAGE_NO = 0;
    static final int MAGIC = 0xB7EE0001;

    /**
     * Tree page layout:
     *   [type:byte][count:short] then `count` fixed-size entries.
     *   Internal entry: [firstKey:long][childPageNo:int]
     *   Leaf entry:     [key:long][heapPageNo:int][slot:short]
     *
     * (A real engine uses a slotted page with line pointers; fixed-size
     *  entries keep this demo short.)
     */
    static final byte LEAF = 0, INTERNAL = 1;
    static final int HEADER = 1 + 2;
    static final int INTERNAL_ENTRY = 8 + 4;
    static final int LEAF_ENTRY = 8 + 4 + 2;

    /** Small fanout so the output is readable. A real 8 KB page holds ~407. */
    static final int FANOUT = 4;

    static final int ROWS_PER_HEAP_PAGE = 70;

    /** One child reference while building: the page number plus the smallest key under it. */
    record Child(long firstKey, int pageNo) {}

    // ------------------------------------------------------------------
    // Building the tree: page numbers are handed out as pages are created.
    // ------------------------------------------------------------------
    static void buildIndex(Pager pager, long rowCount) throws IOException {
        int meta = pager.allocatePage();               // always page 0
        System.out.println("Allocated metapage           -> page " + meta);

        // Level 0: leaf pages. Each gets the next free page number.
        List<Child> level = new ArrayList<>();
        for (long start = 1; start <= rowCount; start += FANOUT) {
            int pageNo = pager.allocatePage();
            ByteBuffer buf = ByteBuffer.allocate(PAGE_SIZE);
            long end = Math.min(start + FANOUT - 1, rowCount);
            buf.put(LEAF).putShort((short) (end - start + 1));
            for (long key = start; key <= end; key++) {
                int heapPage = (int) ((key + ROWS_PER_HEAP_PAGE - 1) / ROWS_PER_HEAP_PAGE);
                short slot = (short) (key - (long) (heapPage - 1) * ROWS_PER_HEAP_PAGE);
                buf.putLong(key).putInt(heapPage).putShort(slot);
            }
            pager.writePage(pageNo, buf);
            level.add(new Child(start, pageNo));
        }
        System.out.printf("Allocated %3d leaf pages     -> pages %d..%d%n",
                level.size(), level.get(0).pageNo(), level.get(level.size() - 1).pageNo());

        // Levels 1, 2, ...: parent pages store their children's page numbers.
        int levels = 1;
        while (level.size() > 1) {
            List<Child> parents = new ArrayList<>();
            for (int i = 0; i < level.size(); i += FANOUT) {
                List<Child> kids = level.subList(i, Math.min(i + FANOUT, level.size()));
                int pageNo = pager.allocatePage();
                ByteBuffer buf = ByteBuffer.allocate(PAGE_SIZE);
                buf.put(INTERNAL).putShort((short) kids.size());
                for (Child c : kids) {
                    buf.putLong(c.firstKey()).putInt(c.pageNo()); // <-- child page number saved here
                }
                pager.writePage(pageNo, buf);
                parents.add(new Child(kids.get(0).firstKey(), pageNo));
            }
            System.out.printf("Allocated %3d level-%d pages  -> pages %d..%d%n",
                    parents.size(), levels, parents.get(0).pageNo(),
                    parents.get(parents.size() - 1).pageNo());
            level = parents;
            levels++;
        }

        // The last page created is the root. Save its number in the metapage.
        int rootPageNo = level.get(0).pageNo();
        ByteBuffer metaBuf = ByteBuffer.allocate(PAGE_SIZE);
        metaBuf.putInt(MAGIC).putInt(rootPageNo).putInt(levels);
        pager.writePage(META_PAGE_NO, metaBuf);
        System.out.println("Root is page " + rootPageNo + " (saved in metapage, page 0)");
    }

    // ------------------------------------------------------------------
    // Searching: start at page 0, then follow page numbers downward.
    // ------------------------------------------------------------------
    static void lookup(Pager pager, long key) throws IOException {
        System.out.println("\nLookup id = " + key);

        ByteBuffer meta = pager.readPage(META_PAGE_NO);
        if (meta.getInt() != MAGIC) throw new IllegalStateException("not an index file");
        int pageNo = meta.getInt();
        int levels = meta.getInt();
        System.out.printf("  read page %-4d offset %-8d metapage: root = page %d, levels = %d%n",
                META_PAGE_NO, Pager.offsetOf(META_PAGE_NO), pageNo, levels);

        while (true) {
            ByteBuffer page = pager.readPage(pageNo);
            byte type = page.get();
            int count = page.getShort();

            if (type == INTERNAL) {
                int child = -1;
                long childKey = 0;
                for (int i = 0; i < count; i++) {
                    long firstKey = page.getLong();
                    int childPageNo = page.getInt();
                    if (key >= firstKey) { child = childPageNo; childKey = firstKey; }
                }
                System.out.printf("  read page %-4d offset %-8d internal: key %d >= %d -> child page %d%n",
                        pageNo, Pager.offsetOf(pageNo), key, childKey, child);
                pageNo = child;
            } else {
                for (int i = 0; i < count; i++) {
                    long k = page.getLong();
                    int heapPage = page.getInt();
                    short slot = page.getShort();
                    if (k == key) {
                        System.out.printf("  read page %-4d offset %-8d leaf: found -> heap page %d, slot %d%n",
                                pageNo, Pager.offsetOf(pageNo), heapPage, slot);
                        return;
                    }
                }
                System.out.printf("  read page %-4d offset %-8d leaf: not found%n",
                        pageNo, Pager.offsetOf(pageNo));
                return;
            }
        }
    }

    public static void main(String[] args) throws IOException {
        Path path = Path.of("users_pkey.idx");
        Files.deleteIfExists(path);

        System.out.println("=== Run 1: build the index ===");
        try (Pager pager = new Pager(path)) {
            buildIndex(pager, 50);
            lookup(pager, 37);
            System.out.println("\nFile size: " + Files.size(path) + " bytes = "
                    + pager.pageCount() + " pages x " + PAGE_SIZE);
        }

        // Simulate a restart: a brand-new Pager with nothing in memory.
        System.out.println("\n=== Run 2: reopen the file (simulated restart) ===");
        try (Pager pager = new Pager(path)) {
            lookup(pager, 37);
        }
    }
}
```

### What each part does

| Part | Role |
|------|------|
| `Pager` | The only class that touches the file. Converts page numbers to offsets and calls `FileChannel.read` / `write`. |
| `allocatePage()` | Assigns the next page number by looking at the current file size. |
| `buildIndex()` | Creates the metapage, then leaf pages, then parent levels, and finally records the root in page 0. |
| `lookup()` | Reads page 0, then follows the stored page numbers down to a leaf. |
| `main()` | Builds the index, searches it, then opens the file again with nothing in memory and searches again. |

### Program output

```text
=== Run 1: build the index ===
Allocated metapage           -> page 0
Allocated  13 leaf pages     -> pages 1..13
Allocated   4 level-1 pages  -> pages 14..17
Allocated   1 level-2 pages  -> pages 18..18
Root is page 18 (saved in metapage, page 0)

Lookup id = 37
  read page 0    offset 0        metapage: root = page 18, levels = 3
  read page 18   offset 147456   internal: key 37 >= 33 -> child page 16
  read page 16   offset 131072   internal: key 37 >= 37 -> child page 10
  read page 10   offset 81920    leaf: found -> heap page 1, slot 37

File size: 155648 bytes = 19 pages x 8192

=== Run 2: reopen the file (simulated restart) ===

Lookup id = 37
  read page 0    offset 0        metapage: root = page 18, levels = 3
  read page 18   offset 147456   internal: key 37 >= 33 -> child page 16
  read page 16   offset 131072   internal: key 37 >= 37 -> child page 10
  read page 10   offset 81920    leaf: found -> heap page 1, slot 37
```

Run 2 starts with an empty memory and still follows exactly the same page numbers, because every number it needs is read back from the file.

---

## 4. What the database actually asks the OS

The program above was run under `strace`, a Linux tool that records every system call a process makes. These are the calls it made on the index file (file descriptor 4), with the data bytes abbreviated:

```text
openat(AT_FDCWD, "users_pkey.idx", O_RDWR|O_CREAT, 0666) = 4

... building: one pwrite64 per page ...
pwrite64(4, <8 KB>, 8192, 147456) = 8192      write page 18 (root)
pwrite64(4, <8 KB>, 8192, 0)      = 8192      write page 0 (metapage: root = 18)

... lookup id = 37 ...
pread64(4, <8 KB>, 8192, 0)       = 8192      read page 0
pread64(4, <8 KB>, 8192, 147456)  = 8192      read page 18
pread64(4, <8 KB>, 8192, 131072)  = 8192      read page 16
pread64(4, <8 KB>, 8192, 81920)   = 8192      read page 10
fsync(4)                          = 0         make sure it is on the device
```

Each call has the form:

```text
pread64(file descriptor, buffer, how many bytes, byte offset in the file)
```

This is the whole interface. The database says *which file* and *which offset*. There is no argument for a disk sector, a block number or a flash address, and the return value is just the byte count. **The OS never tells the database where the data physically is.**

The `fsync` call is the database's way of asking the OS to make sure all written data has actually reached the device, and isn't sitting in the OS's memory cache. Databases call it at commit time.

---

## 5. What the OS knows: the file's map on disk

The OS keeps its own private map for every file. On Linux, `filefrag -v` shows it. For the demo's index file on an ext4 filesystem:

```text
$ filefrag -v users_pkey.idx
Filesystem type is: ef53
File size of users_pkey.idx is 155648 (38 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..      37:    3153216..   3153253:     38:             last,eof
```

This says the file occupies one **extent**, a single continuous run of blocks:

```text
file blocks  0 … 37   →   filesystem blocks 3,153,216 … 3,153,253   (4 KB each)
```

`stat` shows the inode number where ext4 stores this map:

```text
$ stat users_pkey.idx
  Size: 155648      Blocks: 304        IO Block: 4096   regular file
Device: 254,0       Inode: 1204297
```

Inode 1,204,297 lives in ext4's inode table on the disk, and its extent tree holds the line shown by `filefrag`. **This map belongs to the OS.** The database never reads it and never needs to.

### Following one lookup through both maps

Each 8 KB database page covers two 4 KB filesystem blocks. Disk sectors are 512 bytes, so one filesystem block is 8 sectors.

| DB page | File offset | File block (÷ 4,096) | Filesystem block (3,153,216 + file block) | LBA (× 8 sectors) |
|---------|-------------|----------------------|-------------------------------------------|-------------------|
| 0 (meta) | 0          | 0, 1                 | 3,153,216                                 | 25,225,728        |
| 18 (root)| 147,456    | 36, 37               | 3,153,252                                 | 25,226,016        |
| 16       | 131,072    | 32, 33               | 3,153,248                                 | 25,225,984        |
| 10 (leaf)| 81,920     | 20, 21               | 3,153,236                                 | 25,225,888        |

Worked out for the root:

```text
Database:    page 18  →  18 × 8,192 = offset 147,456
Filesystem:  147,456 ÷ 4,096 = file block 36
             extent: file block 0 is at 3,153,216
             3,153,216 + 36 = filesystem block 3,153,252
Device:      3,153,252 × 8 = LBA 25,226,016
```

The database computed only the first line. The OS computed the rest when it handled `pread64(4, …, 147456)`.

> The machine that ran this demo uses a virtual disk (`/dev/vda`), so there is one more translation below the LBA: the hypervisor maps the virtual disk onto real storage. Each extra layer works the same way: it takes an address from above and privately maps it one step down.

---

## 6. What the drive knows

Below the LBA, the operating system's job ends. The drive's own controller chip decides where an LBA physically lives.

### Hard disk (HDD)

An LBA maps to a fixed platter, track and sector, using a formula built into the firmware. When a sector goes bad, the drive moves that LBA to a spare sector and records the change in a **remap table** stored on the drive. Apart from that, an LBA stays in the same physical place for the life of the drive.

### Solid-state drive (SSD)

Flash memory cannot be overwritten in place. A flash page must be erased before it is written again, and erasing only works on large groups of pages. So the SSD uses a **Flash Translation Layer (FTL)**, a lookup table from LBA to flash page, and it updates that table on every write:

```text
The database rewrites page 18 → OS writes LBA 25,226,016

Before:  LBA 25,226,016  ──►  chip 3, erase block 812, flash page 41
After:   LBA 25,226,016  ──►  chip 1, erase block 207, flash page 9
                              (old flash page marked stale, erased later)
```

The physical location of page 18 has moved, but the LBA has not. Nothing above the drive is affected: the filesystem's extent map and the database's page numbers stay exactly the same.

The SSD keeps the FTL table in its own memory and regularly saves it to flash, so it survives a restart. Enterprise SSDs include capacitors that hold enough power to save the table during a sudden power cut.

---

## 7. Are the locations persistent?

| Mapping | Example | Stored where | Who saves it | Same after power-off? |
|---------|---------|--------------|--------------|-----------------------|
| Index name → file | `users_pkey` → `users_pkey.idx` | System catalog (`pg_class` in PostgreSQL) | Database | Yes |
| Root page number | root = page 18 | Metapage, page 0 of the index file | Database | Yes, until the root splits and the database updates it |
| Child page numbers | page 18 → page 16 → page 10 | Entries inside parent pages | Database | Yes |
| Page → file offset | 18 → 147,456 | Not stored: always `page × 8,192` | Database | Yes, it is arithmetic |
| File offset → LBA | block 36 → 3,153,252 | Inode extent tree on the disk | OS filesystem | Yes, unless the OS moves the file (and then it updates the map) |
| LBA → physical (HDD) | LBA → sector | Firmware formula + remap table | Drive | Yes, almost always |
| LBA → physical (SSD) | LBA → flash page | FTL table in the drive | Drive | Table is persistent, but the location changes on every write |

Two mechanisms make these saved mappings safe across a crash:

- **The database's write-ahead log (WAL)** guarantees that a half-finished change, such as a page split that created a new page but hadn't yet updated its parent, is completed or undone during recovery.
- **The filesystem's journal** does the same for the OS's own metadata, such as the extent tree being extended when the file grows.

---

## 8. Answers to the four questions

**1. How does a database point to a physical location on the disk?**

It doesn't. It points to a *page number in a file*. It turns that into a byte offset with `page × 8,192` and passes the file and offset to the OS. The physical location is worked out by the layers below.

**2. Does the operating system give the database those locations?**

No. The OS's filesystem translates file offsets into disk block addresses (LBAs) for its own use, using the map stored in each file's inode. It returns only the data. The database could look at that map with a tool like `filefrag`, but it has no reason to. And even the OS doesn't know the true physical location on an SSD, only the LBA.

**3. Are those locations persistent?**

Yes, at every layer, because each layer saves its own map on the disk. Page numbers and file offsets never change on their own. LBAs change only if the filesystem deliberately moves the file, and it updates its map when it does. On an SSD, the physical flash page changes with every write, but the drive's FTL table records the new place each time.

**4. Does the database engine know where the root page is?**

Yes, but as a page number, not a physical location. Page 0 of the index file (the metapage) stores the root's page number. The database finds the file through its system catalog. From the root, every other page is reached by following page numbers stored inside parent pages.

---

## Side note: databases that skip the filesystem

Some databases can bypass the filesystem and talk to a raw block device directly. Examples are Oracle ASM, and some storage engines that open a whole device with `O_DIRECT`. In that case the database keeps its own map from pages to LBAs, doing the filesystem's job itself. Even then it only reaches the LBA. The drive's firmware still decides the physical location, and on an SSD it still moves the data on every write.

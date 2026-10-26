# Where Does a Database Page Actually Live?

### From B-tree page numbers to physical bits on an HDD or SSD

---

## The question

While studying how a B-tree index finds one row among 10 million, we drew the tree like this:

```text
Root page  →  internal pages 1, 2, … 61  →  leaf pages  →  table pages
```

Every entry in a B-tree page says "the child you want is **page N**". That raises a set of natural questions:

1. How does the database know the exact page number of a page?
2. Is that page number persistent, or is it assigned fresh every time?
3. Is the page number the physical location on the disk?
4. If not, who provides the physical location, and how is the page number mapped to it?
5. After a power-off, is the location the same?
6. If it is, who saved those locations: the database, the OS, or the SSD/HDD itself?

This article answers all six, using one concrete example from start to finish.

---

## Short answer

The page numbers 1, 2 … 61 used earlier were simplified labels. In reality:

- A **page number** is assigned by the database. It is a position inside a file, not a location on disk.
- It is **persistent**: it is written inside the pages themselves, so it survives a reboot.
- Between a page number and the physical bits there are **three translation layers**: the database, the filesystem, and the drive's firmware.
- Each layer owns and saves its own mapping. The page number and the disk address stay the same after a reboot. On an SSD, the physical flash location changes on almost every write, but the drive tracks that itself.

---

## 1. A page number is a position inside a file

A database stores each table and each index as an ordinary **file** managed by the operating system. The database treats that file as an array of fixed-size 8 KB pages:

```text
Index file: base/16384/24576

┌──────────┬──────────┬──────────┬──────────┬─────┬──────────┐
│  page 0  │  page 1  │  page 2  │  page 3  │  …  │ page 290 │  …
└──────────┴──────────┴──────────┴──────────┴─────┴──────────┘
 byte 0     byte 8,192 byte 16,384 …               byte 2,375,680
```

So a page number is just an index into that array, and converting it to a file position is arithmetic:

```text
byte offset in file = page number × 8,192

page 290  →  290 × 8,192 = 2,375,680 bytes from the start of the file
```

### How page numbers get assigned

When the index needs a new page, for example when a full page splits in two, the database adds one more 8 KB page to the end of the file. That page gets the next free number.

Because pages are created in whatever order inserts and splits happen, real page numbers are not tidy. The root might be page 290, and the internal page we called "#47" might be page 1,812. The tree's logical shape and the page numbers have no fixed relationship.

---

## 2. How the database finds the pages

There are two things to find: the file, and the root page inside it. Every other page is reached by following pointers from the root.

### Finding the file

The **system catalog** records which file belongs to which table or index. In PostgreSQL, the `pg_class` catalog stores a file number (`relfilenode`) for every relation:

```text
users_pkey  (index)  →  relfilenode 24576  →  file base/16384/24576
```

### Finding the root page

In PostgreSQL, **page 0 of every B-tree index file is a metapage**. It stores the root's page number, along with the tree's current number of levels:

```text
┌──────────────────────── page 0: METAPAGE ────────────────────────┐
│  root page = 290      tree level of root = 2      ...            │
└──────────────────────────────────────────────────────────────────┘
```

A search always reads page 0 first, jumps to the root, and follows child pointers down from there:

```text
page 0 (metapage) ──► page 290 (root) ──► page 1,812 (internal) ──► page 5,377 (leaf)
                                            child pointers stored inside each parent page
```

When the root splits and a new root is created above it, the database updates the metapage so it points to the new root.

MySQL's InnoDB works the same way in principle. It uses page numbers within a tablespace file, and its data dictionary records the root page number of each index.

---

## 3. Is the page number persistent?

**Yes.** Page numbers are not held in memory and rebuilt at startup. They are written into the pages themselves:

- the **metapage** holds the root's page number,
- every **internal page** holds the page numbers of its children,
- every **leaf page** holds the page numbers of its sibling leaves and the `(table page, slot)` address of each row.

All of this is written to disk, so after a power-off page 290 is still page 290.

There is one risk: what if the power fails halfway through a page split, after the new page is written but before the parent points to it? This is what the **write-ahead log (WAL)** prevents. Every change is first recorded in the log, and during crash recovery the database replays or undoes it, so the tree's pointers are always consistent.

---

## 4. Is the page number the physical location?

**No.** The database never deals with physical locations. It only asks the operating system to "read 8,192 bytes at offset 2,375,680 of file 24576". Two more layers translate that request before it reaches actual hardware.

```text
 ┌───────────────────────────────────────────────────────────────┐
 │ DATABASE        index file 24576, page 290                    │
 └───────────────────────────────┬───────────────────────────────┘
                                 │  page × 8,192
                                 ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ FILE            byte offset 2,375,680 in file 24576           │
 └───────────────────────────────┬───────────────────────────────┘
                                 │  filesystem looks up the file's extents
                                 ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ DISK ADDRESS    LBA 8,393,248 (logical block address)         │
 └───────────────────────────────┬───────────────────────────────┘
                                 │  drive firmware
                                 ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ PHYSICAL        HDD: platter, track, sector                   │
 │                 SSD: NAND chip, erase block, flash page       │
 └───────────────────────────────────────────────────────────────┘
```

### Layer 1: Database (page number → file offset)

- **Who:** the database engine.
- **How:** `offset = page number × 8,192`. No table is needed.
- **Saved where:** the root's number is in the metapage, child numbers are in parent pages, and file names are in the catalog.

### Layer 2: Filesystem (file offset → disk address)

- **Who:** the OS filesystem (ext4, XFS, NTFS, APFS…).
- **How:** a file's data is spread across the disk in runs of blocks called **extents**. The filesystem looks up which extent holds the requested offset and turns it into a **logical block address (LBA)**, which is a plain number naming one block on the drive.
- **Saved where:** in the file's inode and extent tree, which are stored on the disk and protected by the filesystem's own journal.

### Layer 3: Drive firmware (disk address → physical location)

- **Who:** the controller chip inside the HDD or SSD.
- **HDD:** an LBA maps to a fixed platter, track and sector, plus a small table of sectors remapped because they went bad.
- **SSD:** an LBA maps to a flash page through the **Flash Translation Layer (FTL)**, a lookup table that changes constantly (explained in section 6).

---

## 5. A worked example, all the way down

Assume a 4 KB filesystem block size and 512-byte disk sectors.

**Step 1: Database**

```text
Read metapage (page 0)  →  root is page 290
File offset = 290 × 8,192 = 2,375,680 bytes
```

**Step 2: Filesystem**

```text
File block   = 2,375,680 ÷ 4,096 = 580
               (one 8 KB DB page = filesystem blocks 580 and 581)

File's extent: file blocks 0 – 1,023  →  disk blocks 1,048,576 – 1,049,599

Disk block   = 1,048,576 + 580 = 1,049,156
LBA          = 1,049,156 × 8 sectors per block = 8,393,248
```

**Step 3: Drive**

```text
HDD:  LBA 8,393,248  →  a fixed physical sector on a platter
SSD:  FTL table lookup: LBA 8,393,248  →  chip 3, erase block 812, flash page 41
```

Nobody in this chain knows the full path. The database knows page 290, the filesystem knows LBA 8,393,248, and only the drive knows where the bits physically sit.

---

## 6. After a power-off, is the location the same?

It depends on the layer:

| Layer        | Translates                  | Who does it    | Where the mapping is saved                                        | Same after power-off? |
|--------------|-----------------------------|----------------|-------------------------------------------------------------------|-----------------------|
| Database     | page number → file offset   | DB engine      | Arithmetic, plus metapage, parent pages and catalog on disk        | Yes |
| Filesystem   | file offset → LBA           | OS filesystem  | Inode and extent tree on disk, protected by the journal            | Yes, unless the filesystem moves the file and updates its metadata |
| HDD firmware | LBA → physical sector       | Drive controller | Fixed formula, plus a bad-sector remap table                     | Yes, almost always |
| SSD firmware | LBA → physical flash page   | FTL in the SSD controller | Mapping table in the SSD's DRAM, regularly saved to flash | **No:** it changes on every write |

### Why the SSD is different

Flash memory cannot be overwritten in place: a flash page must be erased before it can be written again, and erasing works only on large groups of pages (an erase block). So when the database rewrites page 290, the SSD:

1. writes the new 8 KB of data to a fresh, already-erased flash page,
2. updates its FTL table so LBA 8,393,248 points to that new flash page,
3. marks the old flash page as garbage, to be erased later by background **garbage collection**.

```text
Before update:   LBA 8,393,248  ──►  chip 3, block 812, page 41
After update:    LBA 8,393,248  ──►  chip 1, block 207, page 9     (old page marked stale)
```

The physical location of "page 290" moves with every update, but nothing above the drive notices, because the database, the OS and the filesystem only ever use the LBA, and the LBA does not change.

The SSD controller is responsible for saving its own FTL table so it survives a power loss. Enterprise SSDs include capacitors that hold enough charge to save the table during a sudden power cut.

---

## 7. Who saved the locations?

Each layer saves only its own piece of the mapping:

| What is saved                             | Saved by          | Stored in                                  |
|-------------------------------------------|-------------------|--------------------------------------------|
| Which file belongs to which index         | Database          | System catalog (`pg_class` in PostgreSQL)  |
| Which page is the root                    | Database          | Metapage (page 0 of the index file)        |
| Which page is each child                  | Database          | Entries inside each parent page            |
| Where each file's blocks sit on the drive | Operating system  | Filesystem inode and extent tree           |
| Where each LBA physically sits            | Drive controller  | HDD remap table, or SSD FTL table          |

---

## Key takeaways

- A **page number** is a position inside a database file, assigned by the database when the file grows.
- It is **persistent**, because page numbers are stored inside the pages themselves and protected by the WAL.
- The database finds the **file** through the system catalog and the **root page** through the metapage. Every other page is reached by following child pointers.
- A page number is **not** a physical location. The filesystem maps file offsets to **LBAs**, and the drive maps LBAs to **physical locations**.
- After a reboot, the page number, file offset and LBA are all unchanged. On an SSD the physical flash page keeps moving, but the FTL tracks it, so no layer above the drive ever sees the change.

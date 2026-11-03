# Understanding File Systems in Databases: From Disk Tracks to B+ Trees
- [DBMS NPTEL PPC - IIT KGP](https://www.youtube.com/playlist?list=PLyvBGMFYV3auVdxQ1-88ivNFpmUEy-U3M)
- [NPTEL DBMS Previous Year Questions](https://www.youtube.com/watch?v=mM3wUfPuyGY&list=PL__28a0xFM-_okCioy5J_Z0tRODaJtFrH)
- [DBMS NPTEL ASSIGNMENT WEEK 1-8](https://www.youtube.com/watch?v=z3YUX7Wzlgk&list=PLlI8OFP_hjAeyeqQxHx2ApFbVzcAhTCdJ)
- 


### A step-by-step guide with real data and diagrams

---

## Table of Contents

1. [Basic File System](#1-basic-file-system)
2. [Data Dictionary](#2-data-dictionary)
3. [File Manager, Data Files and Data Dictionary (Disk & DBMS View)](#3-file-manager-data-files-and-data-dictionary)
4. [Visualizing a Track as a Sequence of Blocks](#4-visualizing-a-track-as-a-sequence-of-blocks)
5. [Inside a Track: Track Begin, Blocks, Inter-Block Gaps, Track End](#5-inside-a-track)
6. [How Records Sit Inside a Block](#6-how-records-sit-inside-a-block)
   - 6.1 Fixed-length blocking for fixed-length records
   - 6.2 Variable-length unspanned records
   - 6.3 Variable-length spanned records
   - 6.4 Mapping a real table to a file of attributes and pointers
7. [Sequential File Organization](#7-sequential-file-organization)
8. [Why We Need Reorganization](#8-why-we-need-reorganization)
9. [Indexed Sequential File Organization (ISAM)](#9-indexed-sequential-file-organization-isam)
10. [Multi-Level Sparse Indexing](#10-multi-level-sparse-indexing)
11. [Why We Need Bucketing in Multi-Level Indexing](#11-why-we-need-bucketing-in-multi-level-indexing)
12. [B+ Tree Indexing](#12-b-tree-indexing)
13. [Structure of Leaf and Non-Leaf Nodes (B Tree and B+ Tree)](#13-structure-of-leaf-and-non-leaf-nodes)
14. [Summary Comparison](#14-summary-comparison)

---

## The Running Example

The whole article uses one table, so you can follow the same data from the disk surface up to the index tree.

**Table: `EMPLOYEE`**

| EmpID | Name     | Dept    | Salary |
|-------|----------|---------|--------|
| 101   | Asha     | HR      | 42000  |
| 105   | Bikram   | IT      | 65000  |
| 110   | Chitra   | Finance | 58000  |
| 115   | Deepak   | IT      | 71000  |
| 120   | Esha     | Sales   | 39000  |
| 125   | Farhan   | HR      | 45000  |
| 130   | Gauri    | IT      | 80000  |
| 135   | Harish   | Sales   | 41000  |
| 140   | Ishita   | Finance | 62000  |
| 145   | Jatin    | IT      | 69000  |

**Assumed storage parameters (kept small so the numbers are easy to follow):**

| Parameter | Value |
|-----------|-------|
| Block size (B) | 200 bytes |
| Block pointer size | 4 bytes |
| Record pointer size | 6 bytes |
| Fixed record size (R) | 40 bytes (2 header + 4 EmpID + 20 Name + 10 Dept + 4 Salary; derived in Section 6.1) |



```text
 0    2        6                        26              36       40
 ┌────┬────────┬────────────────────────┬───────────────┬────────┐
 │Hdr │ EmpID  │ Name (20)              │ Dept (10)     │ Salary │
 └────┴────────┴────────────────────────┴───────────────┴────────┘
```
Each field gets that many bytes because of how it’s typed:

- **INT is 4 bytes** in most systems (a 32-bit integer), so EmpID and Salary each take 4 bytes.
- **CHAR(n) is always n bytes.** A short value is padded with blanks: “Asha” is 4 characters but still occupies 20 bytes. That’s what makes the record fixed-length.
- **The 2-byte header is an assumption.** Real systems keep some bookkeeping per record, such as whether it’s deleted and which fields are NULL. Without the header, the record would be 38 bytes.
---

## 1. Basic File System

### 1.1 What is a file?

A **file** is a named collection of related **records** stored on secondary storage (disk/SSD).
A **record** is a collection of **fields** (attributes). A **field** holds one value.

```
FILE  (EMPLOYEE.dat)
 │
 ├── RECORD 1  ──► [ 101 | Asha   | HR      | 42000 ]
 ├── RECORD 2  ──► [ 105 | Bikram | IT      | 65000 ]
 ├── RECORD 3  ──► [ 110 | Chitra | Finance | 58000 ]
 │      ...                 ▲        ▲         ▲
 │                          │        │         │
 │                        FIELD    FIELD     FIELD
 └── RECORD 10 ──► [ 145 | Jatin  | IT      | 69000 ]
```

### 1.2 The storage hierarchy

```
 Logical view (what users see)            Physical view (what the disk stores)
 ─────────────────────────────            ────────────────────────────────────
   Database                                 Disk
     └── Table (EMPLOYEE)                     └── Platter / Surface
           └── Row  (record)                        └── Track
                 └── Column (field)                       └── Sector / Block
                                                                └── Bytes
```

### 1.3 Why files are stored in blocks

The disk never reads or writes a single byte. The smallest unit of transfer between disk and main memory is a **block** (also called a *page*). Reading one record means reading the whole block that contains it.

```
     MAIN MEMORY (buffer)                          DISK
   ┌────────────────────┐   one I/O = one block  ┌──────────┐
   │  Block 2 (200 B)   │ ◄───────────────────── │ Block 2  │
   │  R6 R7 R8 R9 R10   │                        └──────────┘
   └────────────────────┘
```

**Key idea:** the cost of any file operation is measured in **number of block accesses**, because one disk access is thousands of times slower than a memory access. Every file organization in this article tries to reduce block accesses.

### 1.4 Traditional file system vs DBMS

| Aspect | Traditional file system | DBMS |
|--------|-------------------------|------|
| Data redundancy | High (each program keeps its own files) | Controlled |
| Data description | Hard-coded inside programs | Stored centrally in the data dictionary |
| Concurrent access | Difficult | Managed by the transaction manager |
| Security | File-level only | Row/column-level |
| Data independence | None | Logical and physical independence |

---

## 2. Data Dictionary

### 2.1 Definition

A **data dictionary** (also called the *system catalog* or *metadata repository*) is "data about data". It stores the definitions of every object in the database: tables, columns, types, sizes, constraints, indexes, users, and where each file lives on disk.

The DBMS reads the dictionary before executing any query, to know **what** the data looks like and **where** it is.

### 2.2 Example dictionary entries for our table

**SYS_TABLES**

| TableName | FileName      | RecordSize | NumRecords | BlockingFactor | NumBlocks | Organization |
|-----------|---------------|------------|------------|----------------|-----------|--------------|
| EMPLOYEE  | EMPLOYEE.dat  | 40         | 10         | 5              | 2         | Sequential (sorted on EmpID) |


**The blocking factor (bfr)** is the number of records that fit in one block. `(bfr = ⌊ B / R ⌋)`

**SYS_COLUMNS**

| TableName | ColumnName | Position | DataType | Length | Offset | Nullable | Key |
|-----------|------------|----------|----------|--------|--------|----------|-----|
| EMPLOYEE  | EmpID      | 1        | INT      | 4      | 2      | No       | PK  |
| EMPLOYEE  | Name       | 2        | CHAR     | 20     | 6      | No       |     |
| EMPLOYEE  | Dept       | 3        | CHAR     | 10     | 26     | Yes      |     |
| EMPLOYEE  | Salary     | 4        | INT      | 4      | 36     | Yes      |     |
| DEPARTMENT  | DeptID   | 1        | INT      | 4      |        | No       |     |
| DEPARTMENT  | DeptName | 2        | CHAR      | 10    |        | Yes      |     |
| DEPARTMENT  | Location | 3        | CHAR      | 15    |        | Yes      |     |


(Offset 0–1 is a 2-byte record header holding a "deleted" flag — explained in Section 6.)

```text
 SYS_TABLES   → 1 row per table          (EMPLOYEE, DEPARTMENT)
 SYS_COLUMNS  → 1 row per column, all tables together  (4 + 3 = 7 rows)
 SYS_INDEXES  → 1 row per index
```

**SYS_INDEXES**

| IndexName   | TableName | Column | Type           | FileName     | Levels |
|-------------|-----------|--------|----------------|--------------|--------|
| PK_EMP      | EMPLOYEE  | EmpID  | Primary, sparse| PK_EMP.idx   | 1      |
| IDX_EMPDEPT | EMPLOYEE  | Dept   | Secondary      | EMPDEPT.idx  | 1      |

> SQL standard / MySQL: INFORMATION_SCHEMA.COLUMNS <br/>
> PostgreSQL: pg_attribute <br/>
> Oracle: ALL_TAB_COLUMNS <br/>



The request travels down through these layers and the data comes back up the same path. The example follows the article’s lookup: read block 2 of EMPLOYEE.dat to find EmpID 130.
```text
 ═══════════════════════════ USER SPACE ═══════════════════════════════

  ┌────────────────────────── DBMS PROCESS ──────────────────────────┐
  │                                                                  │
  │  Query: SELECT * FROM EMPLOYEE WHERE EmpID = 130                 │
  │                         │                                        │
  │  ┌──────────────────────▼───────────────────────┐                │
  │  │ BUFFER MANAGER                               │                │
  │  │ "Is block 2 of EMPLOYEE.dat in my buffer?"   │── yes → done   │
  │  └──────────────────────┬───────────────────────┘   (no disk I/O)│
  │                         │ no (buffer miss)                       │
  │  ┌──────────────────────▼───────────────────────┐                │
  │  │ FILE MANAGER                                 │                │
  │  │ block (2-1) × 200 B → byte offset 200        │                │
  │  │ call: pread(fd, buf, 200, offset=200)        │                │
  │  └──────────────────────┬───────────────────────┘                │
  └─────────────────────────┼────────────────────────────────────────┘
                            │  ① SYSTEM CALL (trap into kernel)
 ═══════════════════════════▼══════ KERNEL SPACE (OS) ═════════════════

  ┌──────────────────────────────────────────────────────────────────┐
  │ SYSTEM CALL INTERFACE   checks fd, permissions, buffer address   │
  └──────────────────────────────┬───────────────────────────────────┘
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ VFS (Virtual File System)   common layer for all file systems    │
  └──────────────────────────────┬───────────────────────────────────┘
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ PAGE CACHE   "Is this file page already in RAM?" ── yes → copy   │
  └──────────────────────────────┬───────────────────── to buf, ⑧    │
                                 │ no                                │
                                 ▼                                   │
  ┌──────────────────────────────────────────────────────────────────┐
  │ FILE SYSTEM (ext4 / NTFS / XFS)                                  │
  │ ② file offset → physical block, using the inode / extent map:    │
  │    EMPLOYEE.dat, offset 200 → logical block address LBA 81,920   │
  └──────────────────────────────┬───────────────────────────────────┘
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ BLOCK I/O LAYER + I/O SCHEDULER                                  │
  │ ③ builds a request {READ, LBA 81920, 8 sectors}, merges/reorders │
  │    nearby requests to reduce seeks                               │
  └──────────────────────────────┬───────────────────────────────────┘
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ DEVICE DRIVER (SATA/AHCI, SCSI, NVMe)                            │
  │ ④ translates the request into the device's command format        │
  │    e.g. ATA "READ DMA EXT LBA=81920 count=8"                     │
  │    writes it to controller registers / a command queue           │
  └──────────────────────────────┬───────────────────────────────────┘
 ════════════════════════════════▼═══════════ HARDWARE ════════════════
  ┌──────────────────────────────────────────────────────────────────┐
  │ DISK CONTROLLER + FIRMWARE                                       │
  │ ⑤ LBA 81920 → (cylinder, head, sector); seek arm, wait for       │
  │    rotation, read sectors into the drive's cache                 │
  │ ⑥ DMA: copies the data straight into RAM (no CPU copy loop)      │
  │ ⑦ raises an INTERRUPT: "request done"                            │
  └──────────────────────────────┬───────────────────────────────────┘
                                 ▼
          ┌─────────────────────────────────────┐
          │   ●   PLATTER: track / sector       │
          └─────────────────────────────────────┘

 ⑧ RETURN PATH (upward):
    Interrupt handler → driver marks request complete
    → page cache now holds the page → data copied into the DBMS buffer
    → system call returns → file manager finds EmpID 130 in the block
```

### Q1. Why block2?
Because of where EmpID 130 falls in the sorted file. Records are stored in EmpID order, 5 per block (bfr = 5):
```text
 Position:   1    2    3    4    5  │  6    7    8    9   10
 EmpID:     101  105  110  115  120 │ 125  130  135  140  145
            └────── BLOCK 1 ──────┘ └────── BLOCK 2 ──────┘
```
130 is the 7th record. The first 5 records fill block 1, so records 6–10 go into block 2:
```text
 block = ⌈ position / bfr ⌉ = ⌈ 7 / 5 ⌉ = ⌈1.4⌉ = 2
```
The DBMS doesn’t know that 130 is the 7th record, though. It works out the block in one of two ways.

- **With the sparse index (PK_EMP):**
   ```text
    Anchor  Block
     101  →  B1
     125  →  B2
   ```
   The largest anchor ≤ 130 is 125, which points to B2.
- **Without an index, by binary search on the sorted blocks:**
  - Read B1. Its keys run from 101 to 120, and 130 > 120, so the record must be further on.
  - Read B2. Its keys run from 125 to 145, so 130 is in this block.

#### Q1. Why byte offset 200?
Because block 2 starts right after block 1 ends. In the file, the blocks sit one after another, each 200 bytes long:

```text
 EMPLOYEE.dat (byte positions)

 byte 0                200                400
 ┌───────────────────┬───────────────────┐
 │     BLOCK 1       │     BLOCK 2       │
 │ 101 105 110 115   │ 125 130 135 140   │
 │ 120               │ 145               │
 └───────────────────┴───────────────────┘
   bytes 0–199          bytes 200–399
```
The general formula, with blocks numbered from 1 as in the article, is:
```text
 offset = (block number − 1) × block size
        = (2 − 1) × 200 = 200
```

Once the file manager reads those 200 bytes, it finds EmpID 130 inside the block the same way. 130 is the 2nd record in block 2, so it sits at offset 40 within the block (1 × 40), which is byte 200 + 40 = 240 in the file.


#### 2.3 How the dictionary is used

```
 SQL:  SELECT Salary FROM EMPLOYEE WHERE EmpID = 130;

  ┌──────────────┐  1. Does EMPLOYEE exist? Which file?     ┌─────────────────┐
  │ Query        │ ───────────────────────────────────────► │ DATA DICTIONARY │
  │ Processor    │ ◄─────────────────────────────────────── │                 │
  └──────┬───────┘  2. EMPLOYEE.dat, 40-B records,          └─────────────────┘
         │             Salary at offset 36, index PK_EMP
         │
         │  3. Use PK_EMP.idx → block 2 of EMPLOYEE.dat
         ▼
  ┌──────────────┐  4. Read block 2, extract bytes 36..39 of the record
  │ File Manager │ ──────────────────────────────────────► 80000
  └──────────────┘
```

### 2.4 Active vs passive dictionary

| Type | Description |
|------|-------------|
| **Active (integrated)** | Maintained automatically by the DBMS; updated on every `CREATE`/`ALTER`/`DROP`. Always consistent. (e.g. Oracle's catalog, PostgreSQL's `pg_catalog`) |
| **Passive (stand-alone)** | Maintained separately, often manually for documentation. Can go out of date. |

---

## 3. File Manager, Data Files and Data Dictionary

### 3.1 DBMS architecture: where each component sits

```
                    ┌──────────────────────────────────────────────┐
  Users / Apps ───► │               QUERY PROCESSOR                │
                    │  DDL Interpreter │ DML Compiler │ Query Eval │
                    └─────────────────────────┬────────────────────┘
                                              │
                    ┌─────────────────────────▼────────────────────┐
                    │               STORAGE MANAGER                │
                    │ ┌───────────────┐  ┌───────────────────────┐ │
                    │ │ Authorization │  │ Transaction Manager   │ │
                    │ │ & Integrity   │  │ (concurrency/recovery)│ │
                    │ └───────────────┘  └───────────────────────┘ │
                    │ ┌───────────────┐  ┌───────────────────────┐ │
                    │ │ FILE MANAGER  │◄─►│ BUFFER MANAGER        │ │
                    │ │ (space alloc, │  │ (which blocks stay in │ │
                    │ │  block layout)│  │  main memory)         │ │
                    │ └───────┬───────┘  └───────────────────────┘ │
                    └─────────┼────────────────────────────────────┘
                              │ block read / write
   ═══════════════════════════▼═════════════════════════════════════
                         DISK STORAGE
   ┌──────────────┐ ┌─────────────────┐ ┌────────────┐ ┌────────────┐
   │ DATA FILES   │ │ DATA DICTIONARY │ │  INDEXES   │ │ STATISTICS │
   │ EMPLOYEE.dat │ │ SYS_TABLES      │ │ PK_EMP.idx │ │ row counts │
   │ DEPT.dat     │ │ SYS_COLUMNS ... │ │ ...        │ │ histograms │
   └──────────────┘ └─────────────────┘ └────────────┘ └────────────┘
```

### 3.2 Role of each component

| Component | Responsibility | Example in our table |
|-----------|----------------|----------------------|
| **File Manager** | Allocates disk space, maps records to blocks, keeps free-space lists, reads/writes blocks via the OS. | Decides that EmpID 130 lives in block 2, slot 2. |
| **Buffer Manager** | Caches blocks in RAM; decides replacement (LRU etc.). | Keeps block 2 in memory if it is accessed often. |
| **Data Files** | Actual user data. | `EMPLOYEE.dat` with 10 records in 2 blocks. |
| **Data Dictionary** | Metadata. | Record size 40, Salary at offset 36. |
| **Indexes** | Auxiliary files for fast lookup. | `PK_EMP.idx`: (101 → B1), (130 → B2). |
| **Statistical data** | Helps the optimizer choose a plan. | "Dept has 4 distinct values." |

### 3.3 The disk-level view of the same files

```
  Disk (one surface shown)

            Track 0  ──►  [Dictionary blocks D1 D2 ...]
            Track 1  ──►  [EMPLOYEE.dat  B1 B2 ][free ...]
            Track 2  ──►  [PK_EMP.idx    I1    ][EMPDEPT.idx X1]
            Track 3  ──►  [free ............................]
                 ▲
     File Manager keeps a mapping: File → list of (track, block) addresses
```

A file manager typically keeps, per file, a **file header / file control block**:

```
 ┌────────────────────── FILE HEADER: EMPLOYEE.dat ─────────────────────┐
 │ Record format : fixed, 40 bytes                                      │
 │ Blocking factor: 5                                                   │
 │ First block    : (Cyl 0, Track 1, Block 1)                           │
 │ Last block     : (Cyl 0, Track 1, Block 2)                           │
 │ Free-space list: (Track 1, Block 3) → (Track 1, Block 4) → NULL      │
 │ Ordering field : EmpID                                               │
 └──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Visualizing a Track as a Sequence of Blocks

### 4.1 Disk geometry

```
                  Spindle
                     │
          ┌──────────┼──────────┐   ◄── Platter 0, top surface
          │    ╭─────┼─────╮    │
          │   ╭┼─────┼─────┼╮   │   Concentric circles = TRACKS
          │   ││  ╭──┼──╮  ││   │   Track 0 = outermost
          │   ││  │  ●  │  ││   │
          │   ││  ╰─────╯  ││   │   Same track number on every surface
          │   ╰┼───────────┼╯   │   = a CYLINDER
          │    ╰───────────╯    │
          └─────────────────────┘
               ▲
     Read/Write head moves across tracks (seek)
```

### 4.2 "Unrolling" one track

A track is a circle. If you cut it at the **index point** and lay it flat, you get a straight strip of bytes. The file manager divides that strip into blocks.

```
 Circular track                        Same track, unrolled
     ┌───────┐
   ╱  B1  B2  ╲               ┌────┬────┬───┬────┬───┬────┬───┬────┬─────┐
  │ B8      B3 │      ───►    │ TB │ B1 │ G │ B2 │ G │ B3 │ G │... │ TE  │
  │ B7      B4 │              └────┴────┴───┴────┴───┴────┴───┴────┴─────┘
   ╲  B6  B5  ╱                TB = Track Begin, G = Inter-block gap,
     └───────┘                 TE = Track End
       ▲ index point
```

### 4.3 How many blocks fit on a track? (real calculation)

Assume:

- Usable track capacity = **2,000 bytes** (small, for illustration)
- Track-begin overhead (index point + home address + gap) = **100 bytes**
- Block size = **200 bytes**
- Inter-block gap (IBG) = **40 bytes**

```
 Space per block = 200 + 40 = 240 bytes
 Available       = 2000 − 100 = 1900 bytes
 Blocks/track    = ⌊1900 / 240⌋ = 7 blocks
 Wasted at end   = 1900 − 7×240 = 220 bytes  (track end / unused)
```

So our EMPLOYEE file (2 blocks) uses only part of one track.

**Real-world note:** a modern disk has a track of hundreds of kilobytes and 4 KB sectors, but the principle is identical.

---

## 5. Inside a Track

### 5.1 Full layout of one track

```
 ◄──────────────────────────────── ONE TRACK (unrolled) ───────────────────────────────►

 ┌────┬────┬───┬──────────────────┬─────┬──────────────────┬─────┬─────────┬───────┐
 │ IP │ HA │ G │     BLOCK 1      │ IBG │     BLOCK 2      │ IBG │  ....   │  TE   │
 └────┴────┴───┴──────────────────┴─────┴──────────────────┴─────┴─────────┴───────┘
  │     │    │          │             │
  │     │    │          │             └─ Inter-Block Gap: blank space so the head
  │     │    │          │                can finish one block and prepare for the next
  │     │    │          └─ The block: address + (key) + data
  │     │    └─ Gap after the home address
  │     └─ Home Address: track id (cylinder, head), track condition (good/defective)
  └─ Index Point: physical marker where the track begins (Track Begin)

  TE = Track End: remaining space too small for another block
```

### 5.2 Zoom into one block (count-key-data format)

Classic mainframe disks use the **Count-Key-Data (CKD)** format; it shows clearly what metadata a block carries.

```
 ┌─────────────────────────────── BLOCK 1 ──────────────────────────────────┐
 │┌───────────────┐ ┌──┐ ┌──────────┐ ┌──┐ ┌──────────────────────────────┐ │
 ││ COUNT AREA    │ │G │ │ KEY AREA │ │G │ │          DATA AREA           │ │
 ││ Cyl=0 Head=1  │ │  │ │ 120      │ │  │ │ R1 | R2 | R3 | R4 | R5       │ │
 ││ Rec#=1        │ │  │ │(highest  │ │  │ │ (101 105 110 115 120)        │ │
 ││ KeyLen=4      │ │  │ │ key in   │ │  │ │ 5 × 40 = 200 bytes           │ │
 ││ DataLen=200   │ │  │ │ block)   │ │  │ │                              │ │
 │└───────────────┘ └──┘ └──────────┘ └──┘ └──────────────────────────────┘ │
 └──────────────────────────────────────────────────────────────────────────┘
```

| Part | Content | Purpose |
|------|---------|---------|
| Count area | Cylinder, head, record number, key length, data length | Identifies the block and its size |
| Key area | Highest key in the block (here 120) | Lets the controller search a track by key without moving data to memory |
| Data area | The actual records | User data |

### 5.3 Our file on one track

```
 Track 1 (Cylinder 0, Head 1)

 ┌────┬────┬───┬────────────────────────────┬─────┬────────────────────────────┬─────┬─────────────────┬────┐
 │ IP │ HA │ G │ BLOCK 1 (key 120)          │ IBG │ BLOCK 2 (key 145)          │ IBG │ BLOCK 3..7 FREE │ TE │
 │    │0,1 │   │ 101 105 110 115 120        │     │ 125 130 135 140 145        │     │                 │    │
 └────┴────┴───┴────────────────────────────┴─────┴────────────────────────────┴─────┴─────────────────┴────┘
```

### 5.4 Effect of block size on space utilization

| Block size | Blocks/track (1900 usable, IBG 40) | Data bytes on track | Utilization |
|------------|-----------------------------------|---------------------|-------------|
| 100 B | ⌊1900/140⌋ = 13 | 1300 | 68% |
| 200 B | ⌊1900/240⌋ = 7  | 1400 | 74% |
| 400 B | ⌊1900/440⌋ = 4  | 1600 | 84% |
| 900 B | ⌊1900/940⌋ = 2  | 1800 | 95% |

**Lesson:** larger blocks waste less space in gaps, but each transfer moves more data into memory and wastes buffer space when you only need one record. DBMSs balance this (typical 4–16 KB).

---

## 6. How Records Sit Inside a Block

### Key terms

- **Blocking factor (bfr)** = number of records per block = ⌊B / R⌋
- **Unused space per block** = B − (bfr × R)
- **Number of blocks** for r records = ⌈r / bfr⌉

### 6.1 Fixed-Length Blocking for Fixed-Length Records

#### Record format (40 bytes)

```
 Offset: 0    2        6                        26              36       40
         ┌────┬────────┬────────────────────────┬───────────────┬────────┐
         │Hdr │ EmpID  │ Name (CHAR 20)         │ Dept (CHAR 10)│ Salary │
         │ 2B │  4B    │ 20B                    │ 10B           │  4B    │
         └────┴────────┴────────────────────────┴───────────────┴────────┘
  Hdr = 1 byte "deleted" flag + 1 byte "null bitmap"
```

`Name` is padded with blanks: "Asha" occupies 20 bytes = `Asha␣␣␣␣␣␣␣␣␣␣␣␣␣␣␣␣`.

#### Why is the record size 40 bytes?

The record size is simply the **sum of the field sizes plus a small header**:

| Part | Type | Bytes |
|------|------|-------|
| Record header | deleted flag + null bitmap | 2 |
| EmpID | INT | 4 |
| Name | CHAR(20) | 20 |
| Dept | CHAR(10) | 10 |
| Salary | INT | 4 |
| **Total** | | **40** |

```
 R = 2 (header) + 4 (EmpID) + 20 (Name) + 10 (Dept) + 4 (Salary) = 40 bytes
```

- **INT = 4 bytes** in most systems (a 32-bit integer), so EmpID and Salary take 4 bytes each.
- **CHAR(n) always takes n bytes.** A shorter value is padded with blanks: "Asha" is 4 characters but still uses all 20 bytes. This padding is what makes the record *fixed-length*.
- **The 2-byte header is an assumption** for this example. Real systems keep some bookkeeping per record (Is it deleted? Which fields are NULL?). Without the header, R would be 38 bytes.
- **Why these sizes:** they make the arithmetic clean with a 200-byte block, since 200 ÷ 40 = exactly 5 records with 0 bytes left over. With R = 38, a block would still hold ⌊200/38⌋ = 5 records, but 200 − 5×38 = 10 bytes per block would be wasted.

> **Real-world note:** In an actual DBMS the exact size depends on its storage format. Headers are often larger (PostgreSQL's row header is about 23 bytes), and fields may be padded for memory alignment. The method is always the same: add up the field widths plus any overhead.

#### Blocking calculation

```
 bfr          = ⌊200 / 40⌋ = 5 records per block
 unused/block = 200 − 5×40 = 0 bytes
 blocks       = ⌈10 / 5⌉   = 2 blocks
```

#### Block diagram

```
 BLOCK 1 (200 bytes)
 byte 0          40          80          120         160         200
 ┌───────────┬───────────┬───────────┬───────────┬───────────┐
 │ R1        │ R2        │ R3        │ R4        │ R5        │
 │101 Asha   │105 Bikram │110 Chitra │115 Deepak │120 Esha   │
 │HR  42000  │IT  65000  │Fin 58000  │IT  71000  │Sal 39000  │
 └───────────┴───────────┴───────────┴───────────┴───────────┘

 BLOCK 2 (200 bytes)
 ┌───────────┬───────────┬───────────┬───────────┬───────────┐
 │ R6        │ R7        │ R8        │ R9        │ R10       │
 │125 Farhan │130 Gauri  │135 Harish │140 Ishita │145 Jatin  │
 │HR  45000  │IT  80000  │Sal 41000  │Fin 62000  │IT  69000  │
 └───────────┴───────────┴───────────┴───────────┴───────────┘
```

#### Finding a record directly

Because every record is 40 bytes, the address of the i-th record (0-based) is computable:

```
 block number  = ⌊ i / bfr ⌋
 offset inside = ( i mod bfr ) × R

 Record i = 6 (EmpID 130):  block = ⌊6/5⌋ = 1 (i.e. 2nd block)
                            offset = (6 mod 5) × 40 = 40
```

#### Deletion in fixed-length blocks

Set the deleted flag rather than shifting bytes; the slot can be reused.

```
 BLOCK 1 after deleting EmpID 110
 ┌───────────┬───────────┬───────────┬───────────┬───────────┐
 │ R1        │ R2        │ ✗ FREE    │ R4        │ R5        │
 │101 Asha   │105 Bikram │(flag=1)   │115 Deepak │120 Esha   │
 └───────────┴───────────┴─────▲─────┴───────────┴───────────┘
                               └── added to free-slot list
```

| Advantages | Disadvantages |
|------------|---------------|
| Very simple; record address is computable | Wastes space when actual values are shorter (padding "Asha" to 20 bytes) |
| Easy deletion/reuse of slots | Cannot store values longer than the declared size |
| Fast scanning | Wasted space at block end if B is not a multiple of R |

---

### 6.2 Variable-Length Unspanned Records

#### Why variable length?

If `Name` and `Dept` are `VARCHAR`, each record takes only the bytes it needs. Records then have **different sizes**.

#### Variable-length record format

```
 ┌──────────┬────────┬────────┬────────────┬────────────┬──────────────────────┐
 │ RecLen   │ EmpID  │ Salary │ Off(Name)  │ Off(Dept)  │ Name bytes │Dept bytes│
 │ 2B       │ 4B     │ 4B     │ 2B         │ 2B         │ variable   │variable  │
 └──────────┴────────┴────────┴────────────┴────────────┴──────────────────────┘
   fixed-size fields first, then an offset table, then variable data
```

Some systems instead use a **separator** byte (e.g. `$`) between fields, or a length prefix per field.

#### Our records with variable sizes

Assume these record lengths (including headers) for this example:

| Record | EmpID | Name | Length |
|--------|-------|------|--------|
| R1 | 101 | Asha | 30 B |
| R2 | 105 | Bikram | 45 B |
| R3 | 110 | Chitra (with long address field) | 52 B |
| R4 | 115 | Deepak | 38 B |
| R5 | 120 | Esha (with notes field) | 60 B |

Block size here = **120 bytes** (smaller, to make the effect visible).

#### Unspanned rule

> A record must fit **entirely** inside one block. If it does not fit in the remaining space, the remaining space is wasted and the record starts in the next block.

#### Placement step by step

```
 Block 1: R1 (30)            → used 30
          R2 (45)            → used 75
          R3 (52)? 75+52=127 > 120  ✗  → 45 bytes wasted, R3 goes to Block 2

 Block 2: R3 (52)            → used 52
          R4 (38)            → used 90
          R5 (60)? 90+60=150 > 120  ✗  → 30 bytes wasted, R5 goes to Block 3

 Block 3: R5 (60)            → used 60, 60 bytes free (for future inserts)
```

#### Diagram inside the blocks

```
 BLOCK 1 (120 B)
 0         30                75                                     120
 ┌─────────┬─────────────────┬──────────────────────────────────────┐
 │ R1 (30) │ R2 (45)         │░░░░░░░░░░░ WASTED 45 B ░░░░░░░░░░░░░░│
 │101 Asha │105 Bikram       │░░░░░░ (R3 of 52 B can't fit) ░░░░░░░░│
 └─────────┴─────────────────┴──────────────────────────────────────┘

 BLOCK 2 (120 B)
 0                   52              90                              120
 ┌───────────────────┬───────────────┬───────────────────────────────┐
 │ R3 (52)           │ R4 (38)       │░░░░░░░ WASTED 30 B ░░░░░░░░░░░│
 │110 Chitra         │115 Deepak     │░░░░ (R5 of 60 B can't fit) ░░░│
 └───────────────────┴───────────────┴───────────────────────────────┘

 BLOCK 3 (120 B)
 0                             60                                    120
 ┌─────────────────────────────┬─────────────────────────────────────┐
 │ R5 (60)                     │          FREE 60 B                  │
 │120 Esha                     │                                     │
 └─────────────────────────────┴─────────────────────────────────────┘

 Total data = 225 B, total space used = 3 blocks = 360 B, wasted inside = 75 B
```

#### Slotted-page structure (how real DBMSs locate variable records)

```
 ┌────────────────────────────────────────────────────────────────┐
 │ BLOCK HEADER                                                   │
 │ #entries=2 │ free-space end=75 │ slot1:(0,30) │ slot2:(30,45)  │
 ├────────────────────────────────────────────────────────────────┤
 │                     FREE SPACE                                 │
 ├──────────────────────────────────────┬─────────────────────────┤
 │ R2 (Bikram)                          │ R1 (Asha)               │
 └──────────────────────────────────────┴─────────────────────────┘
   Records grow from the end, slot array grows from the front.
   Other structures point to (block, slot#), so a record can move
   inside the block without breaking pointers.
```

| Advantages | Disadvantages |
|------------|---------------|
| Each record is read with a single block access | Wasted space at the end of blocks |
| Simple to implement | Cannot store a record larger than the block |
| No pointers between blocks for one record | Lower space utilization than spanned |

---

### 6.3 Variable-Length Spanned Records

#### Spanned rule

> A record may be **split across blocks**. The first part goes into the remaining space of the current block; a **pointer** at the end of the block points to the block holding the rest.

Block = 120 B, of which **4 B is reserved for the next-block pointer** → 116 B usable for data.

#### Placement step by step (same records)

```
 Block 1 (116 usable):
   R1 (30)          → used 30
   R2 (45)          → used 75
   R3 (52): only 41 B left → store R3 part-1 (41 B), pointer → Block 2

 Block 2:
   R3 part-2 (11 B) → used 11
   R4 (38)          → used 49
   R5 (60)          → used 109  (fits completely)
   7 B free
```

#### Diagram inside the blocks

```
 BLOCK 1 (120 B)
 0         30                75                              116    120
 ┌─────────┬─────────────────┬─────────────────────────────────┬──────┐
 │ R1 (30) │ R2 (45)         │ R3 part-1 (41 B)                │ PTR  │
 │101 Asha │105 Bikram       │110 Chitra … (first 41 bytes)    │ → B2 │
 └─────────┴─────────────────┴─────────────────────────────────┴───┬──┘
                                                                   │
        ┌──────────────────────────────────────────────────────────┘
        ▼
 BLOCK 2 (120 B)
 0      11              49                                  109  116  120
 ┌──────┬───────────────┬──────────────────────────────────┬────┬──────┐
 │R3 p2 │ R4 (38)       │ R5 (60)                          │free│ PTR  │
 │(11 B)│115 Deepak     │120 Esha                          │ 7B │ NULL │
 └──────┴───────────────┴──────────────────────────────────┴────┴──────┘

 Total data = 225 B in 2 blocks (240 B). Only 7 B free + 8 B pointers.
```

Each fragment usually carries a small header: *"continued from block X"* / *"continues in block Y"*, plus a flag saying whether it's the first, middle or last piece.

#### Unspanned vs spanned, side by side

| | Unspanned | Spanned |
|---|-----------|---------|
| Blocks used for our 5 records | 3 | 2 |
| Wasted space | 75 B | 7 B |
| Block accesses to read R3 | 1 | 2 |
| Record larger than a block? | Not possible | Possible |
| Complexity | Low | Higher (pointers, reassembly) |

**Rule of thumb:** use spanned when records are large or vary a lot (e.g. text, images); use unspanned when records are much smaller than a block (usually the case for normal tables).

---

### 6.4 How a Real Table Is Mapped to a File of Attributes and Next Pointers

#### Step 1: Table → record format (from the data dictionary)

```
 EMPLOYEE(EmpID INT, Name VARCHAR(20), Dept CHAR(10), Salary INT)

 Physical record layout (with a next pointer for sequential linking):
 ┌──────┬───────┬────────────────┬────────────┬────────┬──────────────┐
 │ Hdr  │ EmpID │ Name           │ Dept       │ Salary │ NEXT PTR     │
 │ 2B   │ 4B    │ 20B            │ 10B        │ 4B     │ (block,slot) │
 └──────┴───────┴────────────────┴────────────┴────────┴──────────────┘
```

#### Step 2: Row → record → (block, slot)

| Row | EmpID | Stored at (Block, Slot) | Next pointer |
|-----|-------|-------------------------|--------------|
| 1 | 101 | (B1, 0) | (B1, 1) |
| 2 | 105 | (B1, 1) | (B1, 2) |
| 3 | 110 | (B1, 2) | (B1, 3) |
| 4 | 115 | (B1, 3) | (B1, 4) |
| 5 | 120 | (B1, 4) | (B2, 0) |
| 6 | 125 | (B2, 0) | (B2, 1) |
| ... | ... | ... | ... |
| 10 | 145 | (B2, 4) | NULL |

The pair **(block number, slot number)** is the **Record ID (RID)**, sometimes called a *TID* or *ROWID*. Indexes store RIDs.

#### Step 3: Visualization from table to disk

```
   LOGICAL TABLE                    FILE (records)                     DISK BLOCKS
 ┌─────┬────────┬─────┐
 │ 101 │ Asha   │ ... │ ──►  R1 [101|Asha|HR|42000|→R2]  ─┐
 │ 105 │ Bikram │ ... │ ──►  R2 [105|Bikram|IT|...|→R3]  ─┤
 │ 110 │ Chitra │ ... │ ──►  R3 [...|→R4]                ─┼──►  ┌──── BLOCK 1 ────┐
 │ 115 │ Deepak │ ... │ ──►  R4 [...|→R5]                ─┤     │R1 R2 R3 R4 R5   │──┐
 │ 120 │ Esha   │ ... │ ──►  R5 [...|→R6]                ─┘     └─────────────────┘  │
 │ 125 │ Farhan │ ... │ ──►  R6 ...                      ─┐                          │ next
 │ ... │        │     │                                   ├──►  ┌──── BLOCK 2 ────┐  │ block
 │ 145 │ Jatin  │ ... │ ──►  R10 [...|NULL]              ─┘     │R6 R7 R8 R9 R10  │◄─┘
 └─────┴────────┴─────┘                                         └─────────────────┘
```

Two levels of pointers exist in practice:

- **Block-level next pointer:** each block header points to the next block of the file (useful when blocks are not physically contiguous).
- **Record-level next pointer:** each record points to the next record in key order (used in sequential files with overflow, Section 7).

---

## 7. Sequential File Organization

### 7.1 Definition

Records are stored **in sorted order of a key field** (the *ordering key*, here EmpID).

### 7.2 Physical layout

```
 BLOCK 1                              BLOCK 2
 ┌─────┬─────┬─────┬─────┬─────┐     ┌─────┬─────┬─────┬─────┬─────┐
 │ 101 │ 105 │ 110 │ 115 │ 120 │ ──► │ 125 │ 130 │ 135 │ 140 │ 145 │ ──► NULL
 └─────┴─────┴─────┴─────┴─────┘     └─────┴─────┴─────┴─────┴─────┘
        sorted ascending by EmpID, block header holds "next block" pointer
```

### 7.3 Operations

#### (a) Search — binary search on blocks

Find EmpID 130 among *b* = 2 blocks:

```
 Step 1: read middle block (B1): range 101–120. 130 > 120 → go right
 Step 2: read B2: range 125–145 → scan block in memory → found 130
 Cost: ⌈log₂ b⌉ + 1 block accesses (2 here)
```

For a realistic file of 10,000 blocks: log₂(10000) ≈ 14 accesses, versus 5,000 on average for an unsorted file.

#### (b) Sequential read in key order — very fast

Just read B1, B2, … in order: **b accesses, no sorting needed.**

#### (c) Insertion — the hard part

Insert **EmpID 112** (belongs between 110 and 115).

**Option 1 — shift records (expensive):**

```
 Before: B1 [101 105 110 115 120]  B2 [125 130 135 140 145]
 After : B1 [101 105 110 112 115]  B2 [120 125 130 135 140]  B3 [145]
           every later record moves one slot → rewrite half the file on average
```

**Option 2 — overflow area with next pointers (common):**

Each record gets a **next pointer**. The new record goes into a separate **overflow area** and pointers are adjusted so the logical order is preserved.

```
 MAIN FILE (sorted)
 ┌────────────────────────── BLOCK 1 ───────────────────────────┐
 │ [101│→105] [105│→110] [110│→OV1] [115│→120] [120│→125]      │
 └──────────────────────────────────┬───────────────────────────┘
                                    │
                                    ▼
 OVERFLOW AREA (unsorted, append-only)
 ┌──────────────────────────────────┐
 │ OV1: [112 │ Pooja │ IT │ →115]   │───► back to 115 in Block 1
 └──────────────────────────────────┘

 Logical order: 101 → 105 → 110 → 112 → 115 → 120 → ...
```

Insert **EmpID 113** and **111** next:

```
 [110│→OV3]          OV1: [112│→OV2]
                     OV2: [113│→115]
                     OV3: [111│→OV1]

 Logical chain: 110 → 111(OV3) → 112(OV1) → 113(OV2) → 115
```

#### (d) Deletion

Mark the record as deleted (flag = 1) and let the previous record's pointer skip it.

```
 Delete 105:   [101│→110]  [105 ✗ deleted]  [110│→OV3]
```

### 7.4 Advantages and disadvantages

| Advantages | Disadvantages |
|------------|---------------|
| Excellent for batch processing of the whole file in key order (payroll, billing) | Insertion and deletion are expensive (shift) or degrade the file (overflow) |
| Binary search possible: O(log₂ b) | Searching on a non-key field still needs a full scan |
| Simple structure; no extra index file | Overflow chains make searches slower over time |
| Good space utilization | Needs periodic reorganization |

---

## 8. Why We Need Reorganization

### 8.1 What happens to a file over time

After many inserts and deletes, a sequential file looks like this:

```
 MAIN BLOCKS
 ┌─────┬─────┬─────┬─────┬─────┐      ┌─────┬─────┬─────┬─────┬─────┐
 │ 101 │ ✗105│ 110 │ 115 │ ✗120│      │ 125 │ ✗130│ 135 │ 140 │ 145 │
 └─────┴─────┴──┬──┴─────┴─────┘      └─────┴─────┴──┬──┴─────┴─────┘
                │                                     │
 OVERFLOW       ▼                                     ▼
 ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
 │ 112 │ 113 │ 111 │ 137 │ 136 │ 114 │ 138 │ 139 │ ... grows unsorted
 └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

### 8.2 The problems

| Problem | Effect |
|---------|--------|
| **Long overflow chains** | Finding 114 needs: main block → OV(111) → OV(112) → OV(113) → OV(114). Each hop can be a separate block access. |
| **Deleted records (holes)** | Space marked ✗ is wasted but still read during scans. |
| **Loss of physical order** | Records that are logically adjacent are physically scattered, so sequential reads become random reads. |
| **Binary search breaks down** | Binary search works only on the sorted main area; overflow must be scanned. |
| **Overflow area fills up** | No more inserts possible until space is reclaimed. |

### 8.3 What reorganization does

1. Read the main file and overflow area in **logical (pointer) order**.
2. **Drop** records flagged as deleted.
3. **Write** a fresh file in physical sorted order.
4. **Empty** the overflow area and **rebuild** indexes.

```
 AFTER REORGANIZATION
 B1 [101 110 111 112 113]  B2 [114 115 125 135 136]  B3 [137 138 139 140 145]
 Overflow: EMPTY      All pointers again point to the physical next record
```

### 8.4 When to reorganize

- Overflow area exceeds a threshold (e.g. 20% of the main file)
- Average chain length exceeds a limit
- Percentage of deleted records is high
- Typically scheduled at off-peak hours (it locks/rewrites the whole file)

**This cost is exactly why later structures (ISAM, then B+ trees) were developed** — B+ trees reorganize themselves locally during every insert/delete, so a full offline reorganization is never needed.

---

## 9. Indexed Sequential File Organization (ISAM)

### 9.1 Idea

Keep the data file **sorted** (sequential), and add an **index** so we can jump to the right block directly instead of doing binary search over the data file.

**ISAM = Sorted data file + Index + Overflow area.**

### 9.2 Primary sparse index

A **sparse** index has one entry **per block**, not per record. Each entry = (**anchor key**, block pointer), where the anchor is the first key in the block.

Let's extend our file to **4 blocks** (20 records) to make the index more interesting:

```
 Data blocks (sorted):
 B1: 101 105 110 115 120
 B2: 125 130 135 140 145
 B3: 150 155 160 165 170
 B4: 175 180 185 190 195
```

```
   INDEX FILE (sparse)                     DATA FILE (sorted)
 ┌────────────┬──────────┐
 │ Anchor key │ Block ptr│
 ├────────────┼──────────┤
 │    101     │   ──────────────────►  B1 [101 105 110 115 120]
 │    125     │   ──────────────────►  B2 [125 130 135 140 145]
 │    150     │   ──────────────────►  B3 [150 155 160 165 170]
 │    175     │   ──────────────────►  B4 [175 180 185 190 195]
 └────────────┴──────────┘
```

### 9.3 Search for EmpID 160

```
 1. Read the index block. Find the largest anchor ≤ 160 → 150.
 2. Follow the pointer to B3.
 3. Read B3 and scan in memory → found 160.
 Cost = 1 (index) + 1 (data) = 2 block accesses
```

### 9.4 Why a sparse index is small

Index entry = key (4 B) + block pointer (4 B) = 8 B.
Index blocking factor = ⌊200 / 8⌋ = 25 entries per index block.
**One index block can address 25 data blocks = 125 records.**

### 9.5 ISAM structure on disk (cylinder/track index)

Classic IBM ISAM organized the index to match the disk geometry:

```
          MASTER INDEX (optional, in memory)
                     │
          ┌──────────▼──────────┐
          │   CYLINDER INDEX    │   (highest key on each cylinder)
          │ Cyl0: 195  Cyl1: 395│
          └──────┬───────────┬──┘
                 │           │
       ┌─────────▼───┐   ┌───▼─────────┐
       │ TRACK INDEX │   │ TRACK INDEX │   (highest key on each track,
       │  of Cyl 0   │   │  of Cyl 1   │    stored on track 0 of that cylinder)
       │ T1:145 T2:195│  │ ...         │
       └──┬───────┬──┘   └─────────────┘
          │       │
          ▼       ▼
     Track 1   Track 2         ← PRIME DATA AREA (sorted records)
     B1 B2     B3 B4
          │
          ▼
     Cylinder overflow track    ← OVERFLOW AREA
```

### 9.6 Insertion in ISAM with overflow and next pointers

Insert **EmpID 132** — belongs in B2, which is full.

```
 Step 1: Index says 132 belongs to B2 (anchor 125).
 Step 2: B2 is full → push the last record (145) out to overflow,
         keep B2 sorted: B2 = [125 130 132 135 140]
 Step 3: 145 goes to the overflow area; B2's overflow pointer → it.

   INDEX                      PRIME AREA                         OVERFLOW
 ┌─────┬───┐
 │ 101 │ ──────► B1 [101 105 110 115 120]  ovf: NULL
 │ 125 │ ──────► B2 [125 130 132 135 140]  ovf: ──────────► [145 │ NULL]
 │ 150 │ ──────► B3 [150 155 160 165 170]  ovf: NULL
 │ 175 │ ──────► B4 [175 180 185 190 195]  ovf: NULL
 └─────┴───┘
```

Insert **EmpID 142** (also belongs to B2, > 140):

```
 B2 [125 130 132 135 140]  ovf: ──► [142 │ ──► ] ──► [145 │ NULL]
                                     (overflow chain kept in key order)
```

Search for 145 now costs: index (1) + B2 (1) + overflow 142 (1) + overflow 145 (1) = **4 accesses** instead of 2. This degradation is why ISAM also needs **periodic reorganization**.

### 9.7 Dense vs sparse index

```
 DENSE (one entry per record)          SPARSE (one entry per block)
 101 → B1                              101 → B1
 105 → B1                              125 → B2
 110 → B1                              150 → B3
 ... 20 entries                        175 → B4   (4 entries)
```

| | Dense | Sparse |
|---|-------|--------|
| Entries | One per record | One per block |
| Requires sorted data file? | No | **Yes** |
| Size | Large | Small |
| Can answer "does key exist?" from index alone | Yes | No (must read data block) |
| Used for | Secondary indexes | Primary (clustering) index |

### 9.8 Advantages and disadvantages of ISAM

| Advantages | Disadvantages |
|------------|---------------|
| Fast random access (index) **and** fast sequential access (sorted data) | Index is static — built at load time, does not adapt to growth |
| Simple search: few block accesses | Overflow chains degrade performance |
| Good for files with few updates | Needs periodic reorganization |
| | Extra space for index and overflow area |

---

## 10. Multi-Level Sparse Indexing

### 10.1 The problem

When the file is large, the index itself becomes large. Binary search on a big index costs log₂(index blocks) accesses — still too many.

### 10.2 The solution

**Treat the index as a sorted file and build a sparse index on it.** Repeat until the top level fits in **one block**.

### 10.3 Real calculation

Assume a larger EMPLOYEE file:

| Parameter | Value |
|-----------|-------|
| Records (r) | 30,000 |
| Record size | 40 B |
| Block size | 200 B → data bfr = 5 |
| Data blocks | 30,000 / 5 = **6,000** |
| Index entry | key 4 B + pointer 4 B = 8 B |
| Index bfr (fan-out, fo) | ⌊200/8⌋ = **25** |

```
 Level 1 (first-level index): one entry per data block
          entries = 6,000   → blocks = ⌈6000/25⌉ = 240

 Level 2: one entry per level-1 block
          entries = 240     → blocks = ⌈240/25⌉  = 10

 Level 3: one entry per level-2 block
          entries = 10      → blocks = ⌈10/25⌉   = 1   ← TOP LEVEL (stop)

 Number of levels t = ⌈log_fo(6000)⌉ = ⌈log₂₅ 6000⌉ = 3
```

**Search cost = t + 1 = 3 + 1 = 4 block accesses.**

Compare:

| Method | Block accesses |
|--------|----------------|
| Linear scan (unsorted) | 3,000 average |
| Binary search on data file | ⌈log₂ 6000⌉ = 13 |
| Single-level index (binary search on 240 index blocks + 1) | ⌈log₂ 240⌉ + 1 = 9 |
| **Multi-level index** | **4** |

### 10.4 Diagram (with real keys)

```
                         LEVEL 3 (top, 1 block)
               ┌───────────────────────────────────────┐
               │ 101 │ 15101 │ 30101 │ ...  (10 entries)│
               └──┬──────┬─────────────────────────────┘
                  │      │
        ┌─────────┘      └───────────────┐
        ▼                                ▼
   LEVEL 2 (10 blocks)              LEVEL 2 block 2
 ┌──────────────────────────┐     ┌───────────────────────┐
 │ 101 │ 726 │ 1351 │ ...   │     │ 15101 │ 15726 │ ...   │
 │ (25 entries)             │     └───────────────────────┘
 └──┬──────┬────────────────┘
    │      │
    ▼      ▼
   LEVEL 1 (240 blocks)
 ┌────────────────────────────────┐ ┌────────────────────────────┐
 │ 101 │ 126 │ 151 │ ... │ 701    │ │ 726 │ 751 │ ...            │
 │ (25 entries)                   │ └────────────────────────────┘
 └──┬─────┬───────────────────────┘
    │     │
    ▼     ▼
   DATA (6,000 blocks, sorted)
 [101 106 111 116 121] [126 131 136 141 146] ...
```

(Here keys step by 5, so each block holds 5 keys covering a range of 25 key values; level-1 entries step by 25, level-2 entries by 625, and so on.)

### 10.5 Search example: find EmpID 1361

```
 1. Level 3: largest key ≤ 1361 is 101   → go to Level-2 block 1
 2. Level 2: largest key ≤ 1361 is 1351  → go to that Level-1 block
 3. Level 1: largest key ≤ 1361 is 1351  → go to data block [1351 ... 1371]
 4. Data block: scan → 1361 found
 Total: 4 block accesses
```

### 10.6 Advantages and disadvantages

| Advantages | Disadvantages |
|------------|---------------|
| Search cost = number of levels + 1, very small | Static: inserts/deletes are hard because every level is a sorted, packed file |
| Top level is small enough to keep in memory | Inserting can force changes at all levels |
| Works on any sorted file | Still uses overflow areas → reorganization needed |

This static nature is exactly what **B+ trees** fix (Section 12): a B+ tree is a *dynamic* multi-level index.

---

## 11. Why We Need Bucketing in Multi-Level Indexing

### 11.1 The problem: indexing a non-key (duplicate) attribute

Suppose we want an index on **Dept**, which has duplicates:

| EmpID | Dept |
|-------|------|
| 101 | HR |
| 105 | IT |
| 110 | Finance |
| 115 | IT |
| 120 | Sales |
| 125 | HR |
| 130 | IT |
| 135 | Sales |
| 140 | Finance |
| 145 | IT |

The data file is sorted by **EmpID**, not Dept, so records for "IT" are scattered over many blocks. A sparse index is impossible; we need one pointer per record. Three ways to store it:

**Option 1 — repeat the key for every record (dense, duplicates):**

```
 Finance → (B1,2)
 Finance → (B2,3)
 HR      → (B1,0)
 HR      → (B2,0)
 IT      → (B1,1)
 IT      → (B1,3)
 IT      → (B2,1)
 IT      → (B2,4)
 ...
```
✗ Key repeated many times → large index; higher levels have to deal with duplicate keys spanning blocks.

**Option 2 — variable-length index entries (key + list of pointers):**

```
 IT → [(B1,1), (B1,3), (B2,1), (B2,4)]
```
✗ Entries have variable length → index blocks are no longer fixed-format; fan-out is unpredictable; the entry for a popular value may not fit in a block.

**Option 3 — BUCKETING (the standard solution):**

Keep **fixed-length** index entries `(Dept, pointer to bucket)`. Each **bucket** is a block (or chain of blocks) that holds **all record pointers** for that value.

### 11.2 Bucketing diagram

```
   SECONDARY INDEX on Dept        BUCKETS (blocks of record pointers)        DATA FILE (sorted by EmpID)
   (fixed-length entries)

 ┌──────────┬──────┐            ┌──────────────────────┐
 │ Finance  │  ─────────────►   │ (B1,2) (B2,3)        │ ─────┐
 ├──────────┼──────┤            └──────────────────────┘      │
 │ HR       │  ─────────────►   ┌──────────────────────┐      │       B1 [101 105 110 115 120]
 ├──────────┼──────┤            │ (B1,0) (B2,0)        │      ├────►  B2 [125 130 135 140 145]
 │ IT       │  ─────────────►   └──────────────────────┘      │
 ├──────────┼──────┤            ┌──────────────────────┐      │
 │ Sales    │  ─────────────►   │ (B1,1) (B1,3)        │      │
 └──────────┴──────┘            │ (B2,1) (B2,4)        │ ─────┤
                                └──────────────────────┘      │
                                ┌──────────────────────┐      │
                                │ (B1,4) (B2,2)        │ ─────┘
                                └──────────────────────┘
```

If a bucket fills (e.g. "IT" has thousands of employees), it is **chained** to an overflow bucket:

```
 IT ──► [ bucket block 1: 25 RIDs │ next ] ──► [ bucket block 2: 25 RIDs │ next ] ──► NULL
```

### 11.3 Why bucketing matters for *multi-level* indexes

1. **Fixed fan-out:** Every index entry is the same size, so the fan-out stays 25 at every level. Multi-level formulas (Section 10) only work with fixed-size entries.
2. **Each key appears once:** Higher index levels don't need to handle a single key value spread across several lower-level blocks.
3. **Smaller upper levels:** Only distinct values go into the index, so fewer levels.
4. **Easy insertion/deletion:** A new IT employee only appends a RID to the IT bucket; the index levels above don't change.
5. **Fast set operations:** Queries like `Dept = 'IT' AND City = 'Kolkata'` can intersect the two buckets' RID lists before touching data blocks.

### 11.4 Bucketing for overflow in a primary index

The same idea is used in ISAM/multi-level primary indexes: instead of long per-record overflow chains, the overflow records of one data block are grouped into an **overflow bucket** pointed to by that block, so a search reads at most a known number of extra blocks.

### 11.5 Advantages and disadvantages

| Advantages | Disadvantages |
|------------|---------------|
| Fixed-length index entries → predictable fan-out and levels | Extra level of indirection (one more block access) |
| Handles any number of duplicates | Buckets for rare values waste space if a whole block is allocated |
| Inserts/deletes touch only one bucket | Bucket chains for very common values can get long |

---

## 12. B+ Tree Indexing

### 12.1 Why B+ trees?

ISAM and static multi-level indexes degrade with updates. A **B+ tree** is a **dynamic, self-balancing multi-level index**:

- All leaves are at the **same depth** (balanced).
- Nodes are kept **at least half full**.
- Inserts **split** nodes; deletes **merge/redistribute** nodes — locally, so no full reorganization.
- **All data pointers are in the leaves**; leaves are **linked** for sequential access.

It is the default index structure in almost every relational DBMS (MySQL InnoDB, PostgreSQL, Oracle, SQL Server).

### 12.2 Properties for order p (internal) and p_leaf

| Node | Max pointers / keys | Min (except root) |
|------|---------------------|-------------------|
| Internal | p tree pointers, p−1 keys | ⌈p/2⌉ tree pointers |
| Leaf | p_leaf keys with data pointers + 1 next-leaf pointer | ⌈p_leaf/2⌉ keys |
| Root | At least 2 tree pointers (unless it's the only node) | — |

### 12.3 Computing the order from block size (real numbers)

Use realistic values: **block = 512 B**, key (EmpID) = **9 B**, block pointer = **6 B**, record pointer = **7 B**.

```
 Internal node: p·(block ptr) + (p−1)·(key) ≤ 512
                6p + 9(p−1) ≤ 512  →  15p ≤ 521  →  p = 34

 Leaf node:     p_leaf·(key + record ptr) + next ptr ≤ 512
                16·p_leaf + 6 ≤ 512  →  p_leaf = 31
```

Assume nodes are ~69% full on average (typical): internal ≈ 23 pointers, leaf ≈ 21 keys.

```
 Level    Nodes      Pointers / keys
 Root       1         23 pointers
 Level 1   23         529 pointers
 Level 2  529      12,167 pointers
 Leaves  12,167 × 21 keys = 255,507 record pointers
```

**A 3-level tree + leaves indexes ~255,000 records; any search = 4 block accesses (+1 for the data block).**

### 12.4 Building a B+ tree step by step (small order)

To keep diagrams readable, use **order p = 3** (internal: max 3 pointers / 2 keys; leaf: max 2 keys).

Insert EmpIDs in this order: **120, 105, 140, 110, 130, 145, 101, 135**

**Insert 120, 105**

```
 [105 │ 120]          (single leaf = root)
```

**Insert 140 → leaf overflows (3 keys > 2) → split**

Leaf split rule: left keeps ⌈(p_leaf+1)/2⌉ = 2 keys, right gets the rest; **copy** the first key of the right leaf up.

```
            [ 140 ]
           ╱       ╲
   [105 │ 120] ──► [140]
```

**Insert 110 → goes to left leaf [105 120] → overflow → split; copy 110 up**

```
              [ 110 │ 140 ]
             ╱      │      ╲
        [105] ──► [110│120] ──► [140]
```

**Insert 130 → leaf [110 120] → overflow [110 120 130] → split, copy 120 up**
Root would become [110 120 140] (3 keys > 2) → **internal split**: middle key **moves** up (not copied).

```
                      [ 120 ]
                    ╱         ╲
             [ 110 ]           [ 140 ]
            ╱      ╲          ╱       ╲
       [105] ──► [110] ──► [120│130] ──► [140]
```

**Insert 145 → leaf [140] → [140 │ 145]**

```
                      [ 120 ]
                    ╱         ╲
             [ 110 ]           [ 140 ]
            ╱      ╲          ╱       ╲
       [105] ──► [110] ──► [120│130] ──► [140│145]
```

**Insert 101 → leaf [105] → [101 │ 105]**

**Insert 135 → leaf [120 130] → overflow → split into [120 130] and [135]; copy 135 up → internal [135 140]**

```
                         [ 120 ]
                    ╱               ╲
             [ 110 ]                 [ 135 │ 140 ]
            ╱      ╲               ╱       │       ╲
   [101│105] ──► [110] ──► [120│130] ──► [135] ──► [140│145] ──► NULL
       │           │           │            │            │
       ▼           ▼           ▼            ▼            ▼
    data recs   data recs   data recs    data recs    data recs
```

### 12.5 Search: find EmpID 130

```
 Root [120]          : 130 ≥ 120  → right child
 Node [135 │ 140]    : 130 < 135  → leftmost child
 Leaf [120 │ 130]    : found 130 → follow record pointer to data block
 Cost: height (3) + 1 data block = 4
```

### 12.6 Range query: all employees with 110 ≤ EmpID ≤ 140

```
 1. Search for 110 → reach leaf [110]
 2. Follow next-leaf pointers:
    [110] ──► [120│130] ──► [135] ──► [140│145]   stop when key > 140
 Result: 110, 120, 130, 135, 140
```

This leaf chain is the main reason B+ trees beat B trees for databases.

### 12.7 Deletion (brief)

Delete 135 → leaf [135] becomes empty (underflow, min = 1).

- **Redistribute** from a sibling if it has spare keys, or
- **Merge** with a sibling and remove the separator key from the parent (may propagate up; the tree may shrink in height).

```
 After deleting 135 (merge with left sibling [120│130]):
                         [ 120 ]
                    ╱               ╲
             [ 110 ]                 [ 140 ]
            ╱      ╲               ╱       ╲
   [101│105] ──► [110] ──► [120│130] ──► [140│145]
```

### 12.8 B+ tree vs ISAM vs multi-level index

| Feature | ISAM / static multi-level | B+ tree |
|---------|---------------------------|---------|
| Structure | Static | Dynamic, self-balancing |
| Insertion | Overflow chains | Node splits |
| Performance over time | Degrades | Stays O(log n) |
| Reorganization | Periodic full rebuild | Never needed |
| Space utilization | Can be ~100% initially | ~50–100% (avg ~69%) |
| Range queries | Good (sorted data) | Excellent (linked leaves) |
| Concurrency | Simple (static index) | More complex locking |

### 12.9 Advantages and disadvantages of B+ trees

| Advantages | Disadvantages |
|------------|---------------|
| Balanced: every search is the same, predictable cost | Nodes may be only half full → extra space |
| Handles inserts/deletes efficiently without reorganization | Splits/merges add write cost and complexity |
| Very high fan-out → very shallow trees (3–4 levels for millions of rows) | Concurrency control on internal nodes is tricky |
| Linked leaves → fast range and ordered scans | Keys are duplicated (in internal nodes and leaves) |

---

## 13. Structure of Leaf and Non-Leaf Nodes

### 13.1 B Tree (not B+) node — every node stores data pointers

In a **B tree**, each key appears **exactly once** in the whole tree, and **every node** (internal or leaf) carries a data pointer next to each key.

```
 B TREE NODE (order p)

 ┌────┬─────┬─────┬────┬─────┬─────┬────┬─ ─ ─┬───────┬───────┬────┐
 │ P1 │ K1  │ Pr1 │ P2 │ K2  │ Pr2 │ P3 │ ... │ Kq-1  │ Prq-1 │ Pq │
 └─┬──┴─────┴──┬──┴─┬──┴─────┴──┬──┴─┬──┴─ ─ ─┴───────┴───┬───┴─┬──┘
   │           │    │           │    │                     │     │
   ▼           ▼    ▼           ▼    ▼                     ▼     ▼
 subtree    data  subtree     data  subtree               data  subtree
 keys<K1    rec   K1<keys<K2  rec   K2<keys<K3            rec   keys>Kq-1

 Pi  = tree pointer (child node)
 Ki  = search key  (K1 < K2 < ... < Kq-1)
 Pri = data (record) pointer for Ki
 q ≤ p
```

**Example B tree node** with keys from our table:

```
 ┌────┬─────┬────────┬────┬─────┬────────┬────┐
 │ P1 │ 120 │ →(B1,4)│ P2 │ 140 │ →(B2,3)│ P3 │
 └─┬──┴─────┴────────┴─┬──┴─────┴────────┴─┬──┘
   ▼                   ▼                   ▼
 [101 105 110 115]   [125 130 135]       [145 ...]
```

A search for 120 can stop at this internal node.

### 13.2 B+ Tree internal (non-leaf) node — no data pointers

```
 B+ TREE INTERNAL NODE (order p)

 ┌────┬────┬────┬────┬────┬─ ─ ─┬──────┬────┐
 │ P1 │ K1 │ P2 │ K2 │ P3 │ ... │ Kq-1 │ Pq │
 └─┬──┴────┴─┬──┴────┴─┬──┴─ ─ ─┴──────┴─┬──┘
   ▼         ▼         ▼                  ▼
 keys      K1 ≤ keys  K2 ≤ keys         keys
 < K1      < K2       < K3              ≥ Kq-1

 Only keys + child pointers → more keys per block → higher fan-out
```

**Example (from our tree):**

```
 ┌────┬─────┬────┬─────┬────┐
 │ P1 │ 135 │ P2 │ 140 │ P3 │
 └─┬──┴─────┴─┬──┴─────┴─┬──┘
   ▼          ▼          ▼
 [120│130]   [135]     [140│145]
```

**Byte layout (block 512 B, p = 34):**

```
 ┌──────────┬─────┬──────────┬─────┬── ── ──┬─────┬──────────┬─────────┐
 │ P1 (6B)  │K1 9B│ P2 (6B)  │K2 9B│  ...    │K33  │ P34 (6B) │ unused  │
 └──────────┴─────┴──────────┴─────┴── ── ──┴─────┴──────────┴─────────┘
   34×6 + 33×9 = 204 + 297 = 501 bytes used, 11 bytes spare (header)
```

### 13.3 B+ Tree leaf node — data pointers + next-leaf pointer

```
 B+ TREE LEAF NODE

 ┌─────┬──────┬─────┬──────┬─ ─ ─┬──────┬──────┬────────┐
 │ K1  │ Pr1  │ K2  │ Pr2  │ ... │ Kq   │ Prq  │ Pnext  │
 └─────┴──┬───┴─────┴──┬───┴─ ─ ─┴──────┴──┬───┴───┬────┘
          ▼            ▼                   ▼       │
       record       record              record     ▼
       for K1       for K2              for Kq   next leaf (right sibling)
```

**Example (from our tree):**

```
 ┌─────┬─────────┬─────┬─────────┬────────┐       ┌─────┬─────────┬────────┐
 │ 120 │ →(B1,4) │ 130 │ →(B2,1) │ Pnext ─┼─────► │ 135 │ →(B2,2) │ Pnext ─┼──► ...
 └─────┴────┬────┴─────┴────┬────┴────────┘       └─────┴─────────┴────────┘
            ▼               ▼
   [120|Esha|Sales|39000]  [130|Gauri|IT|80000]
```

**Byte layout (block 512 B, p_leaf = 31):**

```
 ┌─────┬──────┬─────┬──────┬── ── ──┬──────┬──────┬───────────┬────────┐
 │K1 9B│Pr1 7B│K2 9B│Pr2 7B│  ...    │K31   │Pr31  │ Pnext 6B  │ unused │
 └─────┴──────┴─────┴──────┴── ── ──┴──────┴──────┴───────────┴────────┘
   31×(9+7) + 6 = 496 + 6 = 502 bytes used
```

In clustered indexes (e.g. MySQL InnoDB primary key), the leaf stores the **whole row** instead of a record pointer.

### 13.4 Node header (in real systems)

Every node block typically starts with a small header:

```
 ┌───────────┬────────────┬───────────┬──────────────┬──────────────┐
 │ Node type │ # of keys  │ Level     │ Parent ptr   │ Prev-leaf ptr│
 │ leaf/int. │ (q)        │ (0=leaf)  │ (optional)   │ (optional)   │
 └───────────┴────────────┴───────────┴──────────────┴──────────────┘
```

### 13.5 B tree vs B+ tree node comparison

| Feature | B Tree | B+ Tree |
|---------|--------|---------|
| Data pointers in internal nodes | Yes | **No** |
| Keys duplicated in leaves | No (each key once) | Yes (separators copied) |
| Leaf linked list | No | **Yes** |
| Fan-out of internal nodes | Lower (data ptrs take space) | **Higher** |
| Tree height for same data | Taller | **Shorter** |
| Search may end at internal node | Yes | No — always reaches a leaf |
| Range queries | Need in-order traversal (back up the tree) | Follow leaf chain |
| Used in practice for DB indexes | Rarely | **Almost always** |

---

## 14. Summary Comparison

### 14.1 File organizations

| Organization | Search (key) | Range / sequential | Insert | Delete | Reorganization |
|--------------|-------------|--------------------|--------|--------|----------------|
| Heap (unordered) | b/2 avg | Full scan + sort | Very fast (append) | Fast (flag) | For space reclaim |
| Sequential (sorted) | log₂ b | Excellent | Slow / overflow | Flag | **Needed** |
| ISAM | levels + 1 (+ overflow) | Good | Overflow chains | Flag | **Needed** |
| Multi-level static index | t + 1 | Good | Hard | Hard | **Needed** |
| B+ tree | height + 1 | Excellent | log n, local splits | log n, local merges | **Not needed** |

### 14.2 Record blocking

| Method | Space usage | Access per record | Large records |
|--------|-------------|-------------------|---------------|
| Fixed-length | Wastes padding | 1 block, computable address | Not allowed |
| Variable unspanned | Wastes block tail | 1 block | Must fit in block |
| Variable spanned | Best | May need 2+ blocks | Allowed |

### 14.3 The big picture

```
 TABLE ──(data dictionary describes)──► RECORD FORMAT
   │
   ▼
 RECORDS ──(blocking: fixed / unspanned / spanned)──► BLOCKS
   │
   ▼
 BLOCKS ──(file manager allocates)──► TRACKS / CYLINDERS on disk
   │
   ▼
 FILE ORGANIZATION: heap → sequential → ISAM → multi-level index → B+ tree
                                 │          │
                         needs reorganization   self-balancing, no reorg
```

### 14.4 Key formulas

| Quantity | Formula |
|----------|---------|
| Blocking factor | bfr = ⌊B / R⌋ |
| Blocks needed | b = ⌈r / bfr⌉ |
| Binary search cost | ⌈log₂ b⌉ |
| Index fan-out | fo = ⌊B / (key + pointer)⌋ |
| Multi-level index levels | t = ⌈log_fo(b₁)⌉ where b₁ = first-level entries |
| Multi-level search cost | t + 1 |
| B+ tree internal order | p·Pb + (p−1)·K ≤ B |
| B+ tree leaf order | p_leaf·(K + Pr) + Pb ≤ B |

---

*End of article.*

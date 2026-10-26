# How a B-Tree Index Speeds Up Database Queries: A Step-by-Step Walkthrough
Imagine a database table containing 10 million users. If you need to retrieve a single user by their ID, how does the database find that row without examining every record?

The answer lies in **database indexes**, particularly B-tree indexes, which allow database engines to locate data efficiently by navigating a tree of sorted keys.

In this article, we will explore how a database stores table rows in pages, why a query without an index can be expensive, how a B-tree index organizes keys, and how an indexed lookup can reduce the number of page accesses dramatically.

To make the process concrete, we will work through a table containing 10 million users and trace a query from the B-tree root to the actual table row.

Reference:
1. [B-Tree page Layout](../tinyDB/btree/btree-page-layout.md)
2. [B-Tree slotted page](../tinyDB/btree/btree-slotted-pages.md)
3. [How Ddatabase Reaches Disk](../tinyDB/how-database-reaches-disk.md)
4. [Cell layout and slotted pages](../tinyDB/btree/cell-layout-and-slotted-pages.md)
5. [Magic numbers bit flags and bit packing](../tinyDB/btree/magic-numbers-bit-flags-and-bit-packing.md)

[Where Does a Database Page Actually Live?](../tinyDB/btree/where-do-page-numbers-live.md)

## 1. Setting Up the Table: Schema and Sample Data
Let's start with a users table containing six columns.
```sql
CREATE TABLE users (
  id         BIGINT,      -- 8 bytes
  name       VARCHAR(50), -- ~20 chars avg
  email      VARCHAR(100),-- ~30 chars avg
  age        INT,         -- 4 bytes
  city       VARCHAR(50), -- ~15 chars avg
  created_at TIMESTAMP    -- 8 bytes
);
```
Assume the table contains 10 million records, with IDs ranging from 1 to 10,000,000.



```
| id        | name         | email                    | age | city      | created_at |
|-----------|--------------|--------------------------|-----|-----------|------------|
| 1         | Aarav Sharma | aarav.sharma@gmail.com   | 29  | Bengaluru | 2024-01-03 |
| 2         | Priya Iyer   | priya.iyer92@yahoo.com  | 34  | Chennai   | 2024-01-03 |
| …         | …            | …                        | …   | …         | …          |
| 10,000,000 | Rohan Mehta | rohan.m@outlook.com      | 41  | Pune      | 2026-09-30 |
```
For this example, assume the rows were inserted in ascending order of id and that the IDs are unique.


Our goal is to execute the following query:
```sql
SELECT * FROM users WHERE id = 7654321;
```
Before introducing an index, let's understand how the database stores the table and how much work is required to find this record.


## 2. Understanding Row Storage: How Much Space Does One Row Occupy?

```text
id          8 bytes
name       21 bytes  (20 chars + 1 length byte)
email      31 bytes  (30 + 1)
age         4 bytes
city       16 bytes  (15 + 1)
created_at  8 bytes
-------------------
Data       88 bytes
Row header +24 bytes (transaction info, null bitmap)
Slot ptr   + 4 bytes (pointer in page's slot array)
-------------------
Total     116 bytes per row
```
[Why is there an extra length byte for characters?]()
**The extra byte represents the length of the string, so the database can determine how much space the value occupies.n**. Actual string storage varies by database engine and row format.

A row also requires metadata, such as transaction information, null tracking, and other row-format overhead.

The slot directory helps the database locate individual rows within a page. Here, we account for its estimated four-byte entry separately from the row data.

> **Estimated effective space per row: 116 bytes.**

## 3. Calculating Rows per Page and Total Table Size
Now that we have an estimated row size, let's calculate how many rows fit on a page.

Assume each page has a 24-byte page header.
```text
Page size          = 8,192 bytes
Page header        =    24 bytes
------------------------------------------
Usable space       = 8,192 − 24 = 8,168 bytes

Effective space per row     =   116 bytes

Rows per page      = 8,168 / 116 = 70.4 → 70 rows

Total pages        = 10,000,000 ÷ 70 = 142,857.1 → 142,858 pages
------------------------------------------
Each page occupies 8 KB, so the estimated table size is:
Table size on disk = 142,858 × 8 KB ≈ 1.09 GB
```
We round down because a page cannot hold a fraction of a row.


### What does the table look like on disk?

If rows are stored in ascending ID order and each page holds exactly 70 rows, the layout would look like this:
```text
Page 1        → IDs 1–70
Page 2        → IDs 71–140
Page 3        → IDs 141–210
...
Page 142,857  → IDs 9,999,921–9,999,990
Page 142,858  → IDs 9,999,991–10,000,000
```
The important point is that the database stores many rows together in pages.

---

### Example: one full page of the users table

```text
┌──────────────────────────────────────────────────────────┐
│          One 8 KB data page (page 1 of 142,858)          │
├──────────────────────────────────────────────────────────┤
│  PAGE HEADER (24 bytes)                                  │
│  Page metadata and row-layout information                │
├──────────────────────────────────────────────────────────┤
│  ROW DATA AREA — 70 rows × 112 bytes = 7,840 bytes       │
│                                                          │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Row: ID 1, "Aarav Sharma", Bengaluru             │   │
│   └──────────────────────────────────────────────────┘   │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Row: ID 2, "Priya Iyer", Chennai                 │   │
│   └──────────────────────────────────────────────────┘   │
│                  ... 67 more rows ...                    │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Row: ID 70, "Kavya Nair", Kochi                  │   │
│   └──────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────┤
│  SLOT DIRECTORY — 70 slots × 4 bytes = 280 bytes         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│   │    Slot 1    │  │    Slot 2    │  │ ... Slot 70  │   │
│   │   Offset A   │  │   Offset B   │  │   Offset Z   │   │
│   └──────────────┘  └──────────────┘  └──────────────┘   │
├──────────────────────────────────────────────────────────┤
│  FREE SPACE — 48 bytes (too small for another row)       │
└──────────────────────────────────────────────────────────┘
```

**Slot directory**

| Slot    | Points to | Row                         |
|---------|-----------|-----------------------------|
| Slot 1  | Offset A  | ID 1, "Aarav Sharma"        |
| Slot 2  | Offset B  | ID 2, "Priya Iyer"          |
| …       | …         | …                           |
| Slot 70 | Offset Z  | ID 70, "Kavya Nair"         |

> A row's address is `(page number, slot number)`. The slot stores the byte offset of the row inside the page, so rows can move within the page without changing their address.

### All pages of the table

```text
┌──────────────────────────────────────────────────────────┐
│      users table on disk: 142,858 pages (~1.09 GB)       │
├──────────────────────────────────────────────────────────┤
│   ┌──────────────────────────────────────────────────┐   │
│   │ Page 1        →  IDs 1 … 70                      │   │
│   └──────────────────────────────────────────────────┘   │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Page 2        →  IDs 71 … 140                    │   │
│   └──────────────────────────────────────────────────┘   │
│                        ...                               │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Page 109,348  →  IDs 7,654,291 … 7,654,360       │   │
│   └──────────────────────────────────────────────────┘   │
│                 ▲ ID 7,654,321 lives here, slot 31       │
│                        ...                               │
│   ┌──────────────────────────────────────────────────┐   │
│   │ Page 142,858  →  IDs 9,999,991 … 10,000,000      │   │
│   └──────────────────────────────────────────────────┘   │
│                 ▲ last page holds only 10 rows           │
└──────────────────────────────────────────────────────────┘
```



## 4. Querying Without an Index: The Cost of a Full Table Scan
Consider the original query again:
```sql
SELECT * FROM users WHERE id = 7654321;
```
Suppose no index exists on id.

The engine doesn’t know where id 7,654,321 lives, so it reads page 1, checks 70 rows, reads page 2, checks 70 rows… until the end (it can’t stop early, because another row could also match).

To retrieve a row from disk, the engine generally needs to access the page containing that row. It does not need to perform a separate disk read for every row on that page.

This page-based storage model is the foundation for understanding query performance.

> The database cannot generally stop after finding the first matching row because, without a uniqueness guarantee or another applicable optimization, additional matching rows may exist later in the table.


> **Cost:** <br/>
> **Total pages scanned ≈ 142,858** <br/>
> **Estimated table data ≈ 1.09 GiB** <br/>

This is a **full table scan**.

Can we avoid scanning all those pages?

## 5. Building the B-tree index on id

![B-Tree Index Lookup Infographic.png](../tinyDB/assets/B-Tree%20Index%20Lookup%20Infographic.png)

A B-tree index maintains keys in sorted order and organizes them into a hierarchy of pages.

Instead of searching every table row, the database navigates this hierarchy to identify the relevant index entry. That entry provides a reference to the corresponding table row, depending on the database's storage architecture.

Let's create the index:
```sql
CREATE INDEX idx_users_id
ON users (id);
```

Conceptually, the index contains entries like these:
```text
ID 1         → Row location
ID 2         → Row location
ID 3         → Row location
...
ID 7,654,321 → Row location
...
ID 10,000,000 → Row location
```
Each entry contains a key and information needed to locate the corresponding row or child page.

The index does not need to store the entire row for every key. In a conventional secondary index on a table stored separately, the index can store the key and a row locator.

### 5.1 Estimating the size of an index entry
Each index entry stores the key plus the row’s location (page, slot):

For our simplified model, assume each index entry requires 20 bytes:
```text
Key (id)            8 bytes
Index tuple header  8 bytes (includes the (page, slot) pointer)
Slot ptr            4 bytes
-------------------------------
Per entry          20 bytes

Usable per index page ≈ 8,192 − 24 header − 16 B-tree footer = 8,152 bytes
Entries per page (fanout) = 8,152 ÷ 20 = 407.6 → 407
```

Assume an index page has 8,192 bytes, a 24-byte page header, and a further 16-byte allowance for other structural overhead.
```text
Page size                   = 8,192 bytes
Page header                 =    24 bytes
Other overhead              =    16 bytes
------------------------------------------
Usable space                 = 8,152 bytes
Entries per page (fanout) = 8,152 / 20 = 407.6 → 407 entries
```
This number is called the **fanout** in our simplified model: approximately how many entries or child pointers a page can accommodate.

Real B-tree pages have different layouts for leaf and internal nodes, and actual fanout depends on key sizes, child pointers, tuple metadata, and page occupancy.


### Estimating the number of index pages
We can now estimate the number of leaf pages needed to store 10 million entries.

Leaf level
```text
Leaf pages:     10,000,000 / 407  = 24,570.02 → 24,571 leaf pages
Internal pages: 24,571 / 407      =     60.37 →     61 pages
Root pages:     61 entries fit in             →      1 page
```

The estimated total index size is:
```text
Height = 3 levels. 
Total index pages = 24,571 + 61 + 1
                  = 24,633 pages

Estimated index size = 24,633 × 8,192 bytes
                     ≈ 192.45 MiB
```
So, under these assumptions, an index of approximately 192 MiB can organize the keys for a table whose estimated data size is approximately 1.09 GiB.

### 5.3 Why three levels are enough
The power of a B-tree comes from its fanout.

A leaf page holds about 407 keys. An internal page can direct the search toward one of approximately 407 lower-level pages. This means an internal page can indirectly cover roughly:

What each level covers:
```text
1 leaf page      covers 407 ids
1 internal page  covers 407 × 407 = 165,649 ids
Root             covers 61 × 165,649 ≈ 10.1M ids ✓
```

## 6. Inside an Indexed Lookup: Finding ID 7,654,321 Step by Step
Read 1 — Start at the Root page. It has 61 entries like:
```text
[1 → internal pg A] [165,650 → pg B] [331,299 → pg C] ...
```
Which child? `(7,654,321 − 1) ÷ 165,649 = 46.2` → child number 47. That child covers ids 7,619,855 to 7,785,503.

**Read 2 — Read 2: Navigate through the internal page:** Internal page #47. It has 407 entries, each pointing to a leaf covering 407 ids.
```text
Offset inside this page = 7,654,321 − 7,619,855 = 34,466
34,466 ÷ 407 = 84.7 → leaf entry number 85
Global leaf number = 46 × 407 + 85 = 18,807
```

**Read 3 — Search the leaf page:** Leaf page #18,807. It covers ids 7,654,043 to 7,654,449. Inside it, a binary search over 407 entries takes about `log₂(407) ≈ 9` comparisons (in RAM, microseconds). It finds:
```text
id 7,654,321 → (heap page 109,348, slot 31)
```
Check: `7,654,321 ÷ 70 = 109,347.4` → page 109,348, and `7,654,321 − 109,347 × 70 = 31` → slot 31 ✓


**Read 4 — Fetch the actual table row:** Table page 109,348. Jump to slot 31, return the row.

Using our assumption of 70 rows per page and 1-based page numbering:
```text
Target ID = 7,654,321
Page number = ceil(7,654,321 / 70) = 109,348

The offset within that page is:
Slot = 7,654,321 − (109,347 × 70) = 31
```

Therefore, the simplified row location is:
```text
Heap page 109,348
Slot 31
```
This page-and-slot calculation assumes perfectly sequential physical placement, with exactly 70 rows per page. Real databases cannot generally derive a row's physical location directly from its ID. **They follow the locator stored in the index**.


## 7. The comparison
| Method           | Pages read | Data read |
|------------------|------------|-----------|
| Full table scan  | 142,858    | ~1.09 GB  |
| B-tree lookup    | 4          | 32 KB     |

The root plus 61 internal pages are only `62 × 8 KB ≈ 496 KB`, so they almost always stay cached in RAM. In practice only the leaf page and the table page may need a disk read: **2 disk I/Os instead of 142,858**.


However, **logical page accesses are not the same as physical disk reads**. A logical access means the database needs a page; a physical read is needed only if the required page is not already available in memory or another applicable cache.

---

# Queries:

## Q1. Why is there an extra length byte for characters?
Imagine storing the name `Aarav Sharma`.

<div style="border:1px solid #64748b;border-radius:10px;padding:9px;font-family:Arial,sans-serif;color:#111827;background-color:#ffffff;">
<h2 style="font-size:18px;line-height:1.2;color:#111827;margin:0 0 9px;font-weight:700;">Conceptual representation of a variable-length string</h2>
<table style="width:100%;border-spacing:5px 0;border-collapse:separate;">
<tr>
<td style="width:30%;background:#dbeafe;color:#111827;border-radius:8px;padding:8px 5px;text-align:center;vertical-align:middle;">
<div style="font-size:16px;font-weight:bold;margin-bottom:5px;">Length</div>
<div style="font-size:22px;color:#1d4ed8;margin-bottom:5px;">12</div>
<div style="font-size:12px;line-height:1.3;color:#475569;">Bytes in this simplified example</div>
</td>
<td style="width:70%;background:#dcf5e5;color:#111827;border-radius:8px;padding:8px 5px;text-align:center;vertical-align:middle;">
<div style="font-size:16px;font-weight:bold;margin-bottom:5px;">String data</div>
<div style="font-size:22px;font-weight:bold;color:#167647;margin-bottom:5px;">Aarav Sharma</div>
<div style="font-size:12px;line-height:1.3;color:#475569;">12 ASCII bytes</div>
</td>
</tr>
</table>
<p style="font-size:11px;line-height:1.3;color:#475569;margin:8px 0 0;">Illustrative layout: a real database may store length information in a different format or keep it in metadata.</p>
</div>

```text
┌──────────────────────────────────────────────────────────┐
│  Conceptual representation of a variable-length string   │
├──────────────────┬───────────────────────────────────────┤
│      LENGTH      │              STRING DATA              │
│                  │                                       │
│        12        │             Aarav Sharma              │
│                  │                                       │
│  Bytes in this   │            12 ASCII bytes             │
│   simplified     │                                       │
│    example       │                                       │
└──────────────────┴───────────────────────────────────────┘
```

**Byte by byte** (`·` is the space character)
```text
 Byte:  1   2   3   4   5   6   7   8   9  10  11  12
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
      │ A │ a │ r │ a │ v │ · │ S │ h │ a │ r │ m │ a │
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

### Why does the database need the length?
Suppose a page contains these values consecutively:
```text
"Ana" | "Christopher" | "Raj"
```
Their lengths differ. The database needs to know where each value ends so it can correctly locate the next value.

For example, if a simplified format uses one length byte:
```text
[03][A][n][a][11][C][h][r][i][s][t][o][p][h][e][r][03][R][a][j]
```

---

## Q2. Why do we need a slot pointer and a slot at all?

First, distinguish the two terms:
- **Slot**: an entry in a page's slot directory that identifies a row's position.
- **Slot pointer** / **slot-directory entry**: the metadata that records where that row is located within the page.

Your article estimates 4 bytes per slot-directory entry. That is a simplified assumption, not a universal size

### Example: three rows inside one page

Suppose a database page contains these rows:

<div style="border:1px solid #e5e7eb; border-radius:12px; padding:16px; background:#fafafa; font-family:sans-serif; color: #000">
<div style="font-weight:600; font-size:18px; margin-bottom:12px;">One 8 KB data page (page 1 of 142,858) </div>
<div style="border:1px solid #a8c1ef; border-radius:8px; padding:12px;">
  <div style="background:#e8f0fe; border-radius:8px; padding:12px 16px; margin-bottom:12px;">
    <div style="color:#1a4fa3; font-weight:600; font-size:12px;">PAGE HEADER (24 bytes)</div>
    <div style="color:#000000;">Page metadata and row-layout information</div>
  </div>

  <div style="border:1px solid #a8c1ef; border-radius:8px; padding:12px; margin-bottom:12px;">
    <div style="color:#000000; margin-bottom:8px; font-weight:600; font-size:12px;">ROW DATA AREA — 70 rows × 112 bytes = 7,840 bytes</div>
    <div style="background:#dff5e6; border-radius:6px; padding:10px 12px; margin-bottom:8px;">Row: ID 1, "Aarav Sharma", Bengaluru</div>
    <div style="background:#dff5e6; border-radius:6px; padding:10px 12px; margin-bottom:8px;">Row: ID 2, "Priya Iyer", Chennai</div>
    <p style="text-align: center">... 67 more rows ... </p>
    <div style="background:#dff5e6; border-radius:6px; padding:10px 12px;">Row: ID 70, "Kavya Nair", Kochi</div>
  </div>

  <div style="background:#f1e8ff; border-radius:8px; padding:12px 16px; margin-bottom:12px;">
    <div style="color:#6b3fa0; font-weight:600; font-size:12px; margin-bottom:10px;">SLOT DIRECTORY — 70 slots × 4 bytes = 280 bytes</div>
    <div style="display:flex; gap:10px;">
      <div style="flex:1; background:#fff; border:1px solid #c9b3ef; border-radius:6px; padding:10px; text-align:center;"><b>Slot 1</b><br><span style="color:#666;">Offset A</span></div>
      <div style="flex:1; background:#fff; border:1px solid #c9b3ef; border-radius:6px; padding:10px; text-align:center;"><b>Slot 2</b><br><span style="color:#666;">Offset B</span></div>
      <div style="flex:1; background:#fff; border:1px solid #c9b3ef; border-radius:6px; padding:10px; text-align:center;"><b>...Slot 70</b><br><span style="color:#666;">Offset Z</span></div>
    </div>
    <div style="color:#555; margin-top:10px;">The entries record where the corresponding row data is located within the page.</div>
  </div>


  <div style="background:#e8f2ff; border-radius:8px; padding:12px; text-align: center">
    <div style="color:#1b8e47; font-weight:600; font-size:12px;">FREE SPACE — 48 bytes (too small for another row) </div>
  </div>
</div>

</div>


### What happens if we don't use slots?
Imagine a page stores rows at fixed, predetermined positions:
```text
Page 10
+-------------------------+
| Row 1: ID 101           |
| Row 2: ID 102           |
| Row 3: ID 103           |
+-------------------------+
```
Now ID 102's name changes from Ana to Christopher, making the row larger.

If rows must stay in fixed positions, the database may have to:
- Move later rows to make room.
- Leave unused gaps if rows cannot be moved.
- Update other references if rows move to different locations.
  With a slot directory, the database can move row data to another suitable location within the page and update the slot's offset. A reference to that slot can remain stable.

```text
Before update:

Slot 1 ──► Row 101 at offset 100
Slot 2 ──► Row 102 at offset 220
Slot 3 ──► Row 103 at offset 340

After moving Row 102:

Slot 1 ──► Row 101 at offset 100
Slot 2 ──► Row 102 at offset 500  ← updated offset
Slot 3 ──► Row 103 at offset 340  ← unchanged
```
The row data moves, but the slot number can stay the same.

Why this is useful
1. Rows can have variable lengths.
2. Space within a page can be reorganized.
3. A slot provides a convenient way to identify a row within its page.
4. The database may be able to update a row's position without changing its slot reference.
   Slots are not mandatory in every possible storage design. They are a common solution for managing variable-length records efficiently.

---

## Q3. Why does ID 7,654,321 → Row location exist in the index?
Consider this SQL query:
```text
SELECT *
FROM users
WHERE id = 7654321;
```
The database must find the user with this ID and return all their columns.

Without an index, the database may need to scan many table pages to find the row. With an index, it can search the sorted IDs and find a reference to the row.

That is what this index entry means:
```text
ID 7,654,321 → Row location
```

It means:
> The key 7,654,321 is associated with information that tells the database where to find the corresponding row.


> **Important**: the index does not inherently know the table page from the ID's numerical value. In a real database, it follows the locator stored in the index. The article's division by 70 is only a simplified illustration assuming perfectly sequential placement and fixed rows per page.
---

## Q4. Why doesn't the index store the entire row?
The index does not need to store the entire row for every key. In a conventional secondary index on a table stored separately, the index can store the key and a row locator.

Suppose the table has six columns
```text
CREATE TABLE users (
    id         BIGINT,
    name       VARCHAR(50),
    email      VARCHAR(100),
    age        INT,
    city       VARCHAR(50),
    created_at TIMESTAMP
);
```

One row might look like this:

| id | name | email | age | city | created\_at |
| --- | --- | --- | --- | --- | --- |
| 7654321 | Rohan Mehta | rohan.m@outlook.com | 41 | Pune | 2026-09-30 |

In the article's simplified storage model, the row occupies approximately 116 bytes, including its estimated overhead.

Now suppose the database builds this index:
```text
CREATE INDEX idx_users_id ON users (id);
```
The index is designed to search by id, not to duplicate every column in the table.

### Option A: Store the complete row in every index entry
Conceptually:
```text
B-tree index
┌─────────────────────────────────────────────┐
│ ID: 7,654,321                               │
│ Name: Rohan Mehta                           │
│ Email: rohan.m@outlook.com                  │
│ Age: 41                                     │
│ City: Pune                                  │
│ Created_at: 2026-09-30                      │
└─────────────────────────────────────────────┘
```
If every index entry stored all the row's data, the index would duplicate much of the table's storage.

Consequences could include:
- A larger index requiring more pages.
- More storage consumption.
- More data to maintain when indexed rows are updated.
- Fewer entries fitting on each index page, potentially increasing tree size.

### Option B: Store the key and row locator
Instead, the secondary index can store something like this:
```text
B-tree index
┌─────────────────────────────────────┐
│ Key: 7,654,321                      │
│ Row locator: (Page 109,348, Slot 31)│
└─────────────────────────────────────┘
                    │
                    │ Follow locator
                    ▼
Table page 109,348
┌─────────────────────────────────────┐
│ Slot 31                             │
│ ID: 7,654,321                       │
│ Name: Rohan Mehta                   │
│ Email: rohan.m@outlook.com          │
│ Age: 41                             │
│ City: Pune                          │
│ Created_at: 2026-09-30              │
└─────────────────────────────────────┘
```
The index entry is much smaller because it contains the search key and a reference to the actual row, rather than all the row's columns.

The database first finds the key, then follows the locator to retrieve the full row.

---

## Q5. What does “secondary index” mean?
A secondary index is an index that is separate from the table's primary storage organization. It provides another way to locate records.

For example:
- An index on id helps find users by ID.
- An index on email helps find users by email.
  Both indexes can refer to the same underlying table rows, so they do not need to duplicate every column.

There is an important distinction: some databases store table rows directly in the primary B-tree, and some indexes can contain additional columns or even cover a query entirely. The key-plus-row-locator design is the conventional secondary-index model described in your article.

---

## Q6. Where page numbers come from?


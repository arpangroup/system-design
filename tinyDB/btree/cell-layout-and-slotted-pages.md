# Cell Layout: What Lives Inside a Slotted Page

### How a database stores keys, child pointers and rows as self-describing records, and why the slot array points to them

---

## Where we left off

Our running example is a `users` table with 10 million rows and a B-tree index on `id`. To find `id = 7,654,321`, the search reads four pages:

```text
Root  →  internal page #47  →  leaf page #18,807  →  table (heap) page #109,348
```

So far each page was described as a **slotted page**:

```text
┌────────┬────────────────┬──────────────┬──────────────────┬─────────┐
│ Header │ Slots (offsets)│  Free space  │     Entries      │ Special │
└────────┴────────────────┴──────────────┴──────────────────┴─────────┘
```

We called the things on the right "entries" and said each one was 16 bytes. This article looks inside those entries. In storage-engine terms, each one is a **cell**, and the rules for how its bytes are arranged are its **cell layout**.

---

## 1. What is a cell?

A **cell** is one record stored inside a page: one key and its pointer in an index page, or one row in a table page.

The slotted page and the cell layout split the work between them:

| Part | Job | Knows |
|------|-----|-------|
| **Slotted page** | Organises the page: where each cell starts, in what order, where free space is | Offsets only. It treats each cell as a blob of bytes |
| **Cell layout** | Organises one record: what kind it is, how long it is, where the key ends and the value starts | Its own bytes only. It doesn't know where it sits in the page |

```text
                           SLOTTED PAGE (the container)
┌────────┬──────┬──────┬──────┬───────────────────┬──────────┬──────────┬──────────┬─────────┐
│ Header │slot 0│slot 1│slot 2│    free space     │  cell    │  cell    │  cell    │ Special │
│        │ →C   │ →A   │ →B   │                   │    A     │    B     │    C     │         │
└────────┴──────┴──────┴──────┴───────────────────┴────┬─────┴──────────┴──────────┴─────────┘
                                                       │
                     ┌─────────────────────────────────┘
                     ▼
               CELL LAYOUT (one record)
         ┌───────┬──────────┬────────────┬───────────┬──────────────┐
         │ flags │ key size │ value size │ key bytes │ value bytes  │
         └───────┴──────────┴────────────┴───────────┴──────────────┘
```

A slot says *where* a cell is. The cell itself says *what* it is and *how big* it is.

---

## 2. Why do we need a cell layout?

A page is just 8,192 raw bytes. A slot is just a 2-byte number, such as `8153`, meaning "a cell starts here". Without a defined layout, the database reading byte 8,153 would have no idea what it is looking at. The cell layout solves five concrete problems.

### Problem 1: Records have different sizes

In the `users` table, `name`, `email` and `city` are `VARCHAR`. "Raj" is 3 bytes; "Christopher" is 11. So rows cannot sit at fixed positions, and the only way to know where a field ends is for the cell to **record its length**.

### Problem 2: Finding the key quickly

A B-tree lookup does a binary search: it follows about 9 slots on a 407-entry page and compares each cell's key with the target. It must reach the key without decoding anything else. A layout with **fixed-size fields first** means the key always starts at the same distance from the start of the cell. In the layout used below that is always `cell offset + 9`, so finding the key is one addition.

### Problem 3: Cells get moved

Cells are copied around constantly:

- **Compaction** squeezes out holes left by deleted cells.
- **Page splits** move half of a page's cells to a new page.
- **Updates** that make a row longer rewrite it somewhere else in the page.

To copy a cell you must know how many bytes to copy. A cell that **records its own size** can be moved by code that knows nothing else about it.

### Problem 4: Different pages need different cells

An internal page needs `key → child page number`. A leaf page needs `key → row address` (or the row itself). The layout defines one format for each, so the same page code can handle both.

### Problem 5: Status information

A cell may need marks such as "deleted" (a tombstone) or "the value continues on an overflow page". A **flags** byte at a fixed position holds these.

> **In short:** the slotted page gives every cell a stable address within the page. The cell layout makes every cell self-describing, so it can be found, compared, copied and moved by generic code.

---

## 3. The two kinds of cells

Most B-tree storage engines use two cell layouts. The ones below follow the generic format described in Alex Petrov's *Database Internals*; real engines use the same ideas with compressed headers (see section 5).

### 3.1 Key cell: used in internal pages

```text
 offset 0          4                 8
        ┌──────────┬─────────────────┬───────────────────────┐
        │ key_size │  child page id  │      key bytes        │
        │ int, 4 B │    int, 4 B     │   key_size bytes      │
        └──────────┴─────────────────┴───────────────────────┘
          fixed-size fields first          variable part last
```

It means: "every key ≥ this key lives under that child page".

### 3.2 Key-value cell: used in leaf pages

```text
 offset 0       1            5              9
        ┌───────┬────────────┬──────────────┬───────────┬──────────────┐
        │ flags │  key_size  │  value_size  │ key bytes │ value bytes  │
        │ 1 B   │  int, 4 B  │   int, 4 B   │ key_size  │  value_size  │
        └───────┴────────────┴──────────────┴───────────┴──────────────┘
                                             ▲            ▲
                                    always at offset 9    at 9 + key_size
```

The value depends on the kind of index:

- **Secondary index** (PostgreSQL-style, our example): the value is the row's address, `(heap page, slot)`.
- **Clustered index** (InnoDB-style): the value is the whole row.

### Why fixed fields come first

If `key_size` came *after* the key, you couldn't find it without already knowing the key's length. Putting every fixed-size field first means their positions are constants, and every variable-length part can be located by adding up sizes:

```text
key starts at    = 9
value starts at  = 9 + key_size
cell ends at     = 9 + key_size + value_size
```

---

## 4. The cells from our example, byte by byte

All numbers are stored big-endian, so the most significant byte comes first.

### 4.1 Internal page #47, entry #85

This entry says: "ids from 7,654,043 onward are in leaf page 18,807".

```text
 byte:   0  1  2  3 │ 4  5  6  7 │ 8  9  10 11 12 13 14 15
       ┌────────────┬────────────┬─────────────────────────┐
 hex:  │00 00 00 08 │00 00 49 77 │00 00 00 00 00 74 ca 9b  │
       ├────────────┼────────────┼─────────────────────────┤
 field:│ key_size   │ child page │ key                     │
 value:│ 8          │ 18,807     │ 7,654,043               │
       └────────────┴────────────┴─────────────────────────┘
                     total: 16 bytes
```

Check the hex: 18,807 = 0x4977 and 7,654,043 = 0x74CA9B. This is the 16-byte internal entry used in the earlier articles.

### 4.2 Leaf page #18,807, the cell for id 7,654,321

This cell says: "id 7,654,321 lives at heap page 109,348, slot 31". The bytes below are printed by the demo program in section 7.

```text
 byte:   0 │ 1  2  3  4 │ 5  6  7  8 │ 9  10 11 12 13 14 15 16 │ 17 18 19 20 │ 21 22
       ┌───┬────────────┬────────────┬─────────────────────────┬─────────────┬───────┐
 hex:  │00 │00 00 00 08 │00 00 00 06 │00 00 00 00 00 74 cb b1  │00 01 ab 24  │00 1f  │
       ├───┼────────────┼────────────┼─────────────────────────┼─────────────┼───────┤
 field:│flg│ key_size   │ value_size │ key                     │ heap page   │ slot  │
 value:│ 0 │ 8          │ 6          │ 7,654,321               │ 109,348     │ 31    │
       └───┴────────────┴────────────┴─────────────────────────┴─────────────┴───────┘
       └──── 9-byte cell header ────┘└─────── key, 8 B ───────┘└──── value, 6 B ─────┘
                                total: 23 bytes
```

Check the hex: 7,654,321 = 0x74CBB1, 109,348 = 0x1AB24, and 31 = 0x1F.

### 4.3 Heap page #109,348, slot 31: the row itself

A table page is also a slotted page, and each row is a cell. Here the "value" is the full row, with one length-prefixed part for each variable-length column. The row is:

| id | name | email | age | city | created_at |
|----|------|-------|-----|------|------------|
| 7,654,321 | Neha Gupta | neha.gupta@gmail.com | 31 | Jaipur | 2025-03-14 |

In PostgreSQL's layout it is stored like this:

```text
┌───────────────────────┬────────┬───────────┬──────────────────────┬──────┬─────────┬─────┬─────────────┐
│ row header            │ id     │ name      │ email                │ age  │ city    │ pad │ created_at  │
│ txn ids, flags,       │ 8 B    │ len 1 B + │ len 1 B +            │ 4 B  │ len 1 B │ 5 B │ 8 B         │
│ null bitmap: 24 B     │        │ 10 B text │ 20 B text            │      │ + 6 B   │     │             │
└───────────────────────┴────────┴───────────┴──────────────────────┴──────┴─────────┴─────┴─────────────┘
 0                       24       32          43                     64     68        75    80          88
```

| Field | Offset in cell | Size | Why |
|-------|----------------|------|-----|
| Row header | 0–23 | 24 B | Transaction info (who created/deleted the row), flags, null bitmap |
| `id` | 24–31 | 8 B | Fixed-size `BIGINT` |
| `name` | 32–42 | 1 + 10 B | Length byte, then "Neha Gupta" |
| `email` | 43–63 | 1 + 20 B | Length byte, then "neha.gupta@gmail.com" |
| `age` | 64–67 | 4 B | Fixed-size `INT`, must start at a multiple of 4 (64 is) |
| `city` | 68–74 | 1 + 6 B | Length byte, then "Jaipur" |
| padding | 75–79 | 5 B | `TIMESTAMP` must start at a multiple of 8 |
| `created_at` | 80–87 | 8 B | Fixed-size `TIMESTAMP` |
| **Total** | | **88 B** | |

To read `email`, the engine starts at 24, skips 8 bytes of `id`, reads `name`'s length byte (11 bytes including the length byte), jumps to 43, and reads `email`'s length byte. The length bytes are what make variable-length columns readable.

> The first article assumed an *average* row of 112 bytes (24 B header + 88 B of data, with typical name and email lengths). Neha's row is shorter than average, at 88 bytes in total. That variation is exactly why rows must carry their own lengths.

---

## 5. Why real engines use smaller headers

The generic leaf cell spends 9 of its 23 bytes on the header: flags, `key_size` and `value_size`. That overhead matters, because it decides how many entries fit in a page, which is the B-tree's fanout.

PostgreSQL's index entry (an `IndexTuple`) packs the same information into 16 bytes:

```text
┌──────────────────────────┬───────────────────┬──────────────────────────┐
│ heap address (TID)       │ size + flags      │ key                      │
│ page 4 B + slot 2 B = 6 B│ 2 B               │ 8 B                      │
└──────────────────────────┴───────────────────┴──────────────────────────┘
                                 total: 16 bytes
```

- The **row address** comes first and is always 6 bytes.
- One **2-byte field** holds both the cell's total size and its flags.
- There is no separate `key_size`, because the column type (`BIGINT` = 8 bytes) is known from the schema.

The effect on our 10-million-row index:

| Layout | Cell | Slot | Per entry | Entries per page | Leaf pages for 10M rows |
|--------|------|------|-----------|------------------|-------------------------|
| Generic (this article) | 23 B | 2 B | 25 B | (8,192 − 24 − 16) ÷ 25 = **326** | 10,000,000 ÷ 326 → **30,675** |
| PostgreSQL | 16 B | 4 B | 20 B | (8,192 − 24 − 16) ÷ 20 = **407** | 10,000,000 ÷ 407 → **24,571** |

Seven bytes saved per cell means 25% more entries per page, and about 6,100 fewer leaf pages (48 MB less) to store and cache. The ideas stay the same: a self-describing cell with fixed fields first. Only the encoding is tighter.

---

## 6. How cells and slots work together

The demo program in section 7 builds part of leaf page #18,807 using the generic leaf cell. Every offset below is real output from that program.

### 6.1 Inserting cells out of order

Keys are inserted in this order: 7,654,**321**, then **043**, then **449**, then **200**. Each insert does two things:

1. **Write the cell** at the end of the free space, so cells are placed right-to-left in *insertion* order.
2. **Insert the cell's offset** into the slot array at the position that keeps the slots in *key* order.

```text
                                                       physical order = insertion order (right to left)
byte 24     26     28     30     32                 8084       8107       8130       8153       8176
   ┌──────┬──────┬──────┬──────┬───────────────────┬──────────┬──────────┬──────────┬──────────┬─────────┐
   │slot 0│slot 1│slot 2│slot 3│    free space     │  cell    │  cell    │  cell    │  cell    │ Special │
   │ 8130 │ 8084 │ 8153 │ 8107 │    8,052 bytes    │ id …200  │ id …449  │ id …043  │ id …321  │  16 B   │
   │ (043)│ (200)│ (321)│ (449)│                   │ 4th      │ 3rd      │ 2nd      │ 1st      │         │
   └──────┴──────┴──────┴──────┴───────────────────┴──────────┴──────────┴──────────┴──────────┴─────────┘
    slots in KEY order                    ▲ freeStart = 32, freeEnd = 8084 ▲
```

Read the slot array left to right and follow each offset: 8130 → …043, 8084 → …200, 8153 → …321, 8107 → …449. That is sorted order, even though the cells themselves are not sorted.

**Why not keep the cells sorted?** Inserting a 23-byte cell in the middle of sorted cells would mean shifting every larger cell along. With slots, the only thing shifted is a few 2-byte offsets in the slot array.

### 6.2 Looking up id 7,654,321

The search is a binary search over **slots**, following each slot to its cell to read the key:

```text
lo = 0, hi = 4
mid = 2  →  slot 2 = 8153  →  key at 8153 + 9  = 7,654,321  ≥ target  →  hi = 2
mid = 1  →  slot 1 = 8084  →  key at 8084 + 9  = 7,654,200  < target  →  lo = 2
lo == hi == 2  →  slot 2 = 8153  →  key 7,654,321 ✓

value at 8153 + 9 + 8 = 8170:  heap page 109,348, slot 31
```

The `+ 9` is the fixed cell-header size from section 3.2. Every probe costs one addition, not a parse. On a full page of 326 cells, the search takes about log₂(326) ≈ 9 probes.

### 6.3 Deleting id 7,654,043

The delete removes **slot 0** and shifts the other slots left. It doesn't touch the cell:

```text
byte 24     26     28     30                        8084       8107       8130       8153       8176
   ┌──────┬──────┬──────┬───────────────────────────┬──────────┬──────────┬──────────┬──────────┬─────────┐
   │slot 0│slot 1│slot 2│       free space          │  cell    │  cell    │ ░░HOLE░░ │  cell    │ Special │
   │ 8084 │ 8153 │ 8107 │       8,054 bytes         │ id …200  │ id …449  │ old …043 │ id …321  │  16 B   │
   │ (200)│ (321)│ (449)│                           │          │          │ 23 B     │          │         │
   └──────┴──────┴──────┴───────────────────────────┴──────────┴──────────┴──────────┴──────────┴─────────┘
```

The 23 bytes at 8130–8152 are now a **hole**. No slot points to them, so they are unreachable, but they aren't part of the free space between `freeStart` and `freeEnd` either. This is **fragmentation**: the page has 8,054 + 23 bytes free, but only 8,054 of them are in one usable block.

### 6.4 Compaction

When fragmentation gets too high (or a new cell won't fit in the main free block), the page is **compacted**: every live cell is copied tightly against the end of the page, and the slots are updated with the new offsets.

```text
byte 24     26     28     30                                   8107       8130       8153       8176
   ┌──────┬──────┬──────┬──────────────────────────────────────┬──────────┬──────────┬──────────┬─────────┐
   │slot 0│slot 1│slot 2│            free space                │  cell    │  cell    │  cell    │ Special │
   │ 8153 │ 8130 │ 8107 │            8,077 bytes               │ id …449  │ id …321  │ id …200  │  16 B   │
   │ (200)│ (321)│ (449)│                                      │          │          │          │         │
   └──────┴──────┴──────┴──────────────────────────────────────┴──────────┴──────────┴──────────┴─────────┘
```

Free space grew from 8,054 to 8,077 bytes: the 23-byte hole was recovered. This only works because each cell records its own size (`9 + key_size + value_size`), so the compaction code knows exactly how many bytes to copy.

### 6.5 The most important consequence: row addresses stay valid

Look at what happened to the cell for 7,654,321 across these steps:

| Step | Cell's byte offset | Slot that points to it |
|------|--------------------|------------------------|
| After inserts | 8153 | slot 2 |
| After compaction | 8130 | slot 1 |

In this sorted index page the slot *position* changed too, because a smaller key was deleted. Index pages are allowed to do that: entries are always found by key, so nothing outside the page refers to a slot number.

**Table pages are different.** Rows on a heap page are *not* kept in key order, so slots never shift. The index leaf says id 7,654,321 lives at `(page 109,348, slot 31)`. If that row is updated and grows, or the page is compacted, the row's cell moves to a new byte offset, but only **slot 31's stored offset** changes:

```text
Before:   slot 31 → offset 2,104  →  [ Neha Gupta row, 88 B ]
After:    slot 31 → offset 1,936  →  [ Neha Gupta row, longer email, 99 B ]

Index entry still says (109,348, 31)  ✓  nothing in the index has to change
```

This is the main reason table rows are addressed as `(page, slot)` and not `(page, byte offset)`. The slot is a level of indirection that lets cells move freely inside the page without breaking any pointer from outside it.

---

## 7. Demo code

`CellDemo.java` builds one leaf page as a slotted page of generic key-value cells and shows each step from section 6. Run it with `java CellDemo.java` (Java 17 or newer).

```java
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.HexFormat;
import java.util.List;

/**
 * A leaf page of the users_pkey index, built as a slotted page of cells.
 *
 * Page layout (8,192 bytes):
 *   [0..23]    header: cellCount (2 B) at 0, freeStart (2 B) at 2, freeEnd (2 B) at 4
 *   [24.. ]    slot array: 2-byte cell offsets, kept sorted by key
 *   ...        free space
 *   [..8175]   cells, added from the end backward
 *   [8176..]   special space (16 B): level, prev page, next page
 *
 * Key-value cell (leaf):
 *   [flags:1][keySize:4][valueSize:4][key bytes][value bytes]
 *   key   = id (8 bytes)
 *   value = heap page (4 bytes) + slot (2 bytes)
 *
 * Run with: java CellDemo.java
 */
public class CellDemo {
    static final int PAGE_SIZE = 8192, HEADER = 24, SPECIAL = 16, SLOT = 2;
    static final int CELL_HEADER = 1 + 4 + 4;
    static final byte FLAG_LIVE = 0x00;

    final ByteBuffer page = ByteBuffer.allocate(PAGE_SIZE);

    CellDemo() {
        setCount(0);
        setFreeStart(HEADER);
        setFreeEnd(PAGE_SIZE - SPECIAL);
    }

    int count()            { return page.getShort(0); }
    int freeStart()        { return page.getShort(2); }
    int freeEnd()          { return page.getShort(4); }
    void setCount(int v)   { page.putShort(0, (short) v); }
    void setFreeStart(int v) { page.putShort(2, (short) v); }
    void setFreeEnd(int v) { page.putShort(4, (short) v); }

    int slot(int i)            { return page.getShort(HEADER + i * SLOT); }
    void setSlot(int i, int o) { page.putShort(HEADER + i * SLOT, (short) o); }

    /** Read only the key of the cell at this offset: skip flags, read keySize, then the key. */
    long keyAt(int cellOffset) {
        return page.getLong(cellOffset + CELL_HEADER);
    }

    int cellSize(int cellOffset) {
        return CELL_HEADER + page.getInt(cellOffset + 1) + page.getInt(cellOffset + 5);
    }

    /** Insert: write the cell at the end of free space, then insert its offset into the sorted slot array. */
    void insert(long id, int heapPage, short heapSlot) {
        int keySize = 8, valueSize = 4 + 2;
        int size = CELL_HEADER + keySize + valueSize;
        if (freeEnd() - freeStart() < size + SLOT) throw new IllegalStateException("page full");

        int cellOffset = freeEnd() - size;
        page.position(cellOffset);
        page.put(FLAG_LIVE).putInt(keySize).putInt(valueSize)
            .putLong(id).putInt(heapPage).putShort(heapSlot);
        setFreeEnd(cellOffset);

        int pos = lowerBound(id);
        for (int i = count(); i > pos; i--) setSlot(i, slot(i - 1)); // shift slots right
        setSlot(pos, cellOffset);
        setCount(count() + 1);
        setFreeStart(HEADER + count() * SLOT);
    }

    /** Binary search over the slots; each probe follows a slot to its cell and reads the key. */
    int lowerBound(long id) {
        int lo = 0, hi = count();
        while (lo < hi) {
            int mid = (lo + hi) >>> 1;
            if (keyAt(slot(mid)) < id) lo = mid + 1; else hi = mid;
        }
        return lo;
    }

    String find(long id) {
        int i = lowerBound(id);
        if (i == count() || keyAt(slot(i)) != id) return "not found";
        int off = slot(i);
        int heapPage = page.getInt(off + CELL_HEADER + 8);
        short heapSlot = page.getShort(off + CELL_HEADER + 12);
        return "slot index " + i + " -> cell at offset " + off
                + " -> (heap page " + heapPage + ", slot " + heapSlot + ")";
    }

    /** Delete: remove the slot. The cell's bytes stay where they are until compaction. */
    void delete(long id) {
        int i = lowerBound(id);
        for (int j = i; j < count() - 1; j++) setSlot(j, slot(j + 1));
        setCount(count() - 1);
        setFreeStart(HEADER + count() * SLOT);
    }

    /** Compaction: rewrite all live cells tightly against the end of the page. */
    void compact() {
        List<byte[]> cells = new ArrayList<>();
        for (int i = 0; i < count(); i++) {
            int off = slot(i);
            byte[] c = new byte[cellSize(off)];
            page.get(off, c);
            cells.add(c);
        }
        int end = PAGE_SIZE - SPECIAL;
        for (int i = 0; i < cells.size(); i++) {
            byte[] c = cells.get(i);
            end -= c.length;
            page.put(end, c);
            setSlot(i, end);
        }
        setFreeEnd(end);
    }

    void print(String title) {
        System.out.println(title);
        System.out.printf("  header: count=%d, freeStart=%d, freeEnd=%d, free=%d bytes%n",
                count(), freeStart(), freeEnd(), freeEnd() - freeStart());
        for (int i = 0; i < count(); i++) {
            int off = slot(i);
            System.out.printf("  slot %d @ byte %d -> cell @ %d..%d  key %,d%n",
                    i, HEADER + i * SLOT, off, off + cellSize(off) - 1, keyAt(off));
        }
        System.out.println();
    }

    public static void main(String[] args) {
        CellDemo leaf = new CellDemo();

        // Inserted out of key order on purpose.
        leaf.insert(7_654_321L, 109_348, (short) 31);
        leaf.insert(7_654_043L, 109_344, (short) 33);
        leaf.insert(7_654_449L, 109_350, (short) 19);
        leaf.insert(7_654_200L, 109_346, (short) 50);
        leaf.print("After 4 inserts (321, 043, 449, 200):");

        int off = leaf.slot(leaf.lowerBound(7_654_321L));
        byte[] cell = new byte[leaf.cellSize(off)];
        leaf.page.get(off, cell);
        System.out.println("Cell bytes for id 7,654,321 at offset " + off + ":");
        System.out.println("  " + HexFormat.ofDelimiter(" ").formatHex(cell));
        System.out.println();

        System.out.println("find(7,654,321): " + leaf.find(7_654_321L));
        System.out.println();

        leaf.delete(7_654_043L);
        leaf.print("After deleting 7,654,043 (its 23-byte cell at 8130..8152 is now an unreachable hole):");

        leaf.compact();
        leaf.print("After compaction:");
    }
}
```

### Output

```text
After 4 inserts (321, 043, 449, 200):
  header: count=4, freeStart=32, freeEnd=8084, free=8052 bytes
  slot 0 @ byte 24 -> cell @ 8130..8152  key 7,654,043
  slot 1 @ byte 26 -> cell @ 8084..8106  key 7,654,200
  slot 2 @ byte 28 -> cell @ 8153..8175  key 7,654,321
  slot 3 @ byte 30 -> cell @ 8107..8129  key 7,654,449

Cell bytes for id 7,654,321 at offset 8153:
  00 00 00 00 08 00 00 00 06 00 00 00 00 00 74 cb b1 00 01 ab 24 00 1f

find(7,654,321): slot index 2 -> cell at offset 8153 -> (heap page 109348, slot 31)

After deleting 7,654,043 (its 23-byte cell at 8130..8152 is now an unreachable hole):
  header: count=3, freeStart=30, freeEnd=8084, free=8054 bytes
  slot 0 @ byte 24 -> cell @ 8084..8106  key 7,654,200
  slot 1 @ byte 26 -> cell @ 8153..8175  key 7,654,321
  slot 2 @ byte 28 -> cell @ 8107..8129  key 7,654,449

After compaction:
  header: count=3, freeStart=30, freeEnd=8107, free=8077 bytes
  slot 0 @ byte 24 -> cell @ 8153..8175  key 7,654,200
  slot 1 @ byte 26 -> cell @ 8130..8152  key 7,654,321
  slot 2 @ byte 28 -> cell @ 8107..8129  key 7,654,449
```

### Mapping the code to the concepts

| Code | Concept |
|------|---------|
| `CELL_HEADER = 1 + 4 + 4` | The fixed-size part of the cell: flags, key size, value size |
| `keyAt(off)` → `getLong(off + CELL_HEADER)` | Fixed fields first, so the key is always at `+9` |
| `cellSize(off)` | The cell describes its own size, which is what makes copying possible |
| `insert()`: write at `freeEnd − size` | Cells grow from the end of the page backward |
| `insert()`: shift slots right | Slots are kept in key order; cells are not |
| `lowerBound()` | Binary search over slots, dereferencing each one to read its key |
| `delete()` | Removes the slot only, leaving a hole |
| `compact()` | Copies live cells together, using each cell's size, and rewrites the slots |

---

## 8. When a cell is too big: overflow

A cell must fit in one page. If a value is too large, for example a long `bio` column of several kilobytes, the cell keeps only the beginning of the value and a pointer to **overflow pages** that hold the rest, with its flags byte marking it as overflowed:

```text
Leaf page                                        Overflow page 52,019       Overflow page 52,020
┌──────────────────────────────────────────┐    ┌───────────────────────┐   ┌───────────────────────┐
│ flags = OVERFLOW │ sizes │ key │ first   │    │ next = 52,020         │   │ next = none           │
│ part of value │ overflow page = 52,019 ──┼───►│ … next 8 KB of value ─┼──►│ … rest of value       │
└──────────────────────────────────────────┘    └───────────────────────┘   └───────────────────────┘
```

This keeps cells small enough that every page still holds many of them, which keeps the fanout high. PostgreSQL's version of this is called **TOAST**: values larger than about 2 KB are compressed and/or moved to a separate table, and the row keeps only a small pointer.

---

## Summary

| Concept | What it is | Why it exists |
|---------|------------|---------------|
| **Slotted page** | Header + sorted slot array + free space + cells | Gives each cell a stable in-page address and keeps keys in order without moving cells |
| **Slot** | A 2–4 byte offset pointing to a cell | Lets cells be placed in any physical order, and moved, while the order and addresses stay stable |
| **Cell** | One self-describing record: header + key + value | Lets generic code find the key, compare it, and copy the record without a schema |
| **Key cell** | `key_size, child page, key` | Internal pages: routes the search to a child page |
| **Key-value cell** | `flags, key_size, value_size, key, value` | Leaf pages: maps a key to a row address or the row itself |
| **Row cell** | Row header + fields, with length bytes for variable columns | Table pages: stores one row of any size |
| **Overflow** | Part of a value moved to other pages | Keeps cells small so pages keep a high fanout |

The relationship in one line: **the slotted page manages *where* cells are; the cell layout manages *what* is inside each cell.** Together they let a page hold variable-length records in key order, find any one of them with a binary search, and reorganise itself without breaking any pointer from outside the page.

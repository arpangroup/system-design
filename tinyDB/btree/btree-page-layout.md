# B-tree lookup for id 7,654,321, page by page

Each box below is one 8 KB page, drawn with its real sections. The search reads one page per level, following the matching entry downward.

```sql
SELECT * FROM users WHERE id = 7654321;
```

## The four pages on the search path

```text
                         ┌──────────────────────────────────────────────────────────────┐
                         │     ROOT PAGE  ·  level 2  ·  covers all 10,000,000 ids      │
                         ├──────────────────────────────────────────────────────────────┤
                         │  PAGE HEADER  (24 B)                                         │
                         │    LSN, checksum, flags, pd_lower, pd_upper                  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  LINE POINTERS  (61 × 4 B = 244 B)                           │
                         │    [1] [2] … [46] [47] [48] … [61]                           │
                         │    sorted by key; each holds an entry's byte offset          │
                         ├──────────────────────────────────────────────────────────────┤
                         │  FREE SPACE  (6,932 B)                                       │
                         │    pointers grow ↓    ·    entries grow ↑                    │
                         ├──────────────────────────────────────────────────────────────┤
                         │  INDEX ENTRIES  (61 × 16 B = 976 B)   key → child page       │
                         │    #1    1            →  internal page 1                     │
                         │    …                                                         │
                         │    #46   7,454,206    →  internal page 46                    │
                         │    #47   7,619,855    →  internal page 47   ◄ MATCH          │
                         │    #48   7,785,504    →  internal page 48                    │
                         │    …                                                         │
                         │    #61   9,938,941    →  internal page 61                    │
                         ├──────────────────────────────────────────────────────────────┤
                         │  SPECIAL SPACE  (16 B)                                       │
                         │    level 2  ·  prev: none  ·  next: none                     │
                         └──────────────────────────────────────────────────────────────┘
                                                         │  entry #47
                                                         │
                                                         ▼
                         ┌──────────────────────────────────────────────────────────────┐
                         │ INTERNAL PAGE #47  ·  level 1  ·  ids 7,619,855 … 7,785,503  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  PAGE HEADER  (24 B)                                         │
                         │    LSN, checksum, flags, pd_lower, pd_upper                  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  LINE POINTERS  (407 × 4 B = 1,628 B)                        │
                         │    [1] [2] … [84] [85] [86] … [407]                          │
                         ├──────────────────────────────────────────────────────────────┤
                         │  FREE SPACE  (12 B)                                          │
                         │    page is full                                              │
                         ├──────────────────────────────────────────────────────────────┤
                         │  INDEX ENTRIES  (407 × 16 B = 6,512 B)   key → leaf page     │
                         │    #1    7,619,855    →  leaf 18,723                         │
                         │    …                                                         │
                         │    #84   7,653,636    →  leaf 18,806                         │
                         │    #85   7,654,043    →  leaf 18,807        ◄ MATCH          │
                         │    #86   7,654,450    →  leaf 18,808                         │
                         │    …                                                         │
                         │    #407  7,785,097    →  leaf 19,129                         │
┌────────────────────┐   ├──────────────────────────────────────────────────────────────┤   ┌────────────────────┐
│    Internal #46    │   │  SPECIAL SPACE  (16 B)                                       │   │    Internal #48    │
│  ids 7,454,206 …   │◄──│    level 1  ·  prev: internal 46  ·  next: internal 48       │──►│  ids 7,785,504 …   │
└────────────────────┘   └──────────────────────────────────────────────────────────────┘   └────────────────────┘
                                                         │  entry #85
                                                         │
                                                         ▼
                         ┌──────────────────────────────────────────────────────────────┐
                         │ LEAF PAGE #18,807  ·  level 0  ·  ids 7,654,043 … 7,654,449  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  PAGE HEADER  (24 B)                                         │
                         │    LSN, checksum, flags, pd_lower, pd_upper                  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  LINE POINTERS  (407 × 4 B = 1,628 B)                        │
                         │    [1] [2] … [278] [279] [280] … [407]                       │
                         ├──────────────────────────────────────────────────────────────┤
                         │  FREE SPACE  (12 B)                                          │
                         │    page is full                                              │
                         ├──────────────────────────────────────────────────────────────┤
                         │  INDEX ENTRIES  (407 × 16 B = 6,512 B)   key → (page, slot)  │
                         │    #1    7,654,043    →  (109,344, 33)                       │
                         │    …                                                         │
                         │    #278  7,654,320    →  (109,348, 30)                       │
                         │    #279  7,654,321    →  (109,348, 31)     ◄ FOUND           │
                         │    #280  7,654,322    →  (109,348, 32)                       │
                         │    …                                                         │
                         │    #407  7,654,449    →  (109,350, 19)                       │
┌────────────────────┐   ├──────────────────────────────────────────────────────────────┤   ┌────────────────────┐
│    Leaf #18,806    │   │  SPECIAL SPACE  (16 B)                                       │   │    Leaf #18,808    │
│  ids 7,653,636 …   │◄──│    level 0  ·  prev: leaf 18,806  ·  next: leaf 18,808       │──►│  ids 7,654,450 …   │
└────────────────────┘   └──────────────────────────────────────────────────────────────┘   └────────────────────┘
                                                         │  entry #279 → (109,348, 31)
                                                         │
                                                         ▼
                         ┌──────────────────────────────────────────────────────────────┐
                         │   TABLE (HEAP) PAGE #109,348  ·  ids 7,654,291 … 7,654,360   │
                         ├──────────────────────────────────────────────────────────────┤
                         │  PAGE HEADER  (24 B)                                         │
                         │    LSN, checksum, flags, pd_lower, pd_upper                  │
                         ├──────────────────────────────────────────────────────────────┤
                         │  LINE POINTERS  (70 × 4 B = 280 B)                           │
                         │    [1] [2] … [30] [31] [32] … [70]                           │
                         ├──────────────────────────────────────────────────────────────┤
                         │  FREE SPACE  (48 B)                                          │
                         │    too small for another 116 B row                           │
                         ├──────────────────────────────────────────────────────────────┤
                         │  ROW DATA  (70 × 112 B = 7,840 B)                            │
                         │    slot 1    7,654,291  Ishaan Verma   Delhi                 │
                         │    …                                                         │
                         │    slot 30   7,654,320  Meera Pillai   Kochi                 │
                         │    slot 31   7,654,321  Neha Gupta     Jaipur      ◄ ROW     │
                         │    slot 32   7,654,322  Arjun Rao      Hyderabad             │
                         │    …                                                         │
                         │    slot 70   7,654,360  Sneha Das      Kolkata               │
                         ├──────────────────────────────────────────────────────────────┤
                         │  NO SPECIAL SPACE  (0 B)                                     │
                         │    heap pages don't need sibling links                       │
                         └──────────────────────────────────────────────────────────────┘
```
### Why is the root at level 2? 
B-tree levels are counted **from the bottom up**. Leaf pages are level 0, the internal pages above them are level 1, and the root is level 2. Counting from the bottom is used because leaves never change level: when the root splits and a new root is added on top, only that new page gets a new number.

Counting from the bottom is used because leaves stay at level 0 forever. When the root splits and a new root is added on top, only the new root gets a new number; no existing page has to be renumbered.

### Why does the root have 61 line pointers?
A page has **one line pointer per entry**, and the root has one entry for each internal page below it. There are 24,571 leaf pages, and each internal page can point to 407 of them, so 24,571 ÷ 407 = 60.37, which rounds up to 61 internal pages. That means 61 entries in the root, and so 61 line pointers.
```text
Leaf pages     = 10,000,000 ÷ 407 = 24,570.02 → 24,571
Internal pages = 24,571 ÷ 407     = 60.37     → 61
```
61 internal pages → 61 entries in the root → **61 line pointers**. The same rule gives 407 line pointers in a full internal or leaf page, and 70 in a table page holding 70 rows.

```text
byte 0                                                              byte 8,191
┌────────┬─────────────────┬──────────────┬───────────────────┬─────────┐
│ Header │ Line pointers ─►│  FREE SPACE  │◄─ Entries / rows  │ Special │
└────────┴─────────────────┴──────────────┴───────────────────┴─────────┘
```

Every page diagram marks where `pd_lower` and `pd_upper` sit. Lines under the page connect each line pointer (#1, the matching one, and the last) to the entry it points to. Entry #1 is drawn nearest the end of the page, because entries are added from the end backward. Under each page there is a one-line sum showing how its free space was calculated.

## What each section of a page does

| Section        | Grows from        | Purpose                                                                 |
|----------------|-------------------|-------------------------------------------------------------------------|
| Page header    | fixed, at start   | Page metadata, plus where free space starts (`pd_lower`) and ends (`pd_upper`) |
| Line pointers  | start → downward  | One 4-byte slot per entry, kept in key order, holding that entry's byte offset |
| Free space     | middle            | The gap between line pointers and entries; new data fills it from both sides |
| Entries / rows | end → upward      | The actual index entries (key → pointer) or table rows                  |
| Special space  | fixed, at end     | B-tree pages only: tree level and links to the previous and next page   |

> Entries are physically stored from the end of the page backward, often out of key order. The line pointers are what keep them sorted, so a binary search walks the line pointers, not the raw bytes.

## Byte budget of each page

| Page                  | Header | Line pointers  | Free  | Entries / rows    | Special | Total     |
|-----------------------|--------|----------------|-------|-------------------|---------|-----------|
| Root                  | 24     | 61 × 4 = 244   | 6,932 | 61 × 16 = 976     | 16      | **8,192** |
| Internal #47          | 24     | 407 × 4 = 1,628 | 12   | 407 × 16 = 6,512  | 16      | **8,192** |
| Leaf #18,807          | 24     | 407 × 4 = 1,628 | 12   | 407 × 16 = 6,512  | 16      | **8,192** |
| Heap page #109,348    | 24     | 70 × 4 = 280   | 48    | 70 × 112 = 7,840  | 0       | **8,192** |

> The root is mostly empty because only 61 internal pages exist below it. It fills up as the table grows, and when it has no room left it splits and the tree gains a level.

## How each key in the diagram was calculated

**Root entries** — entry *k* points to internal page *k*, which starts at:

```text
first id = (k − 1) × 165,649 + 1

#46  →  45 × 165,649 + 1 = 7,454,206
#47  →  46 × 165,649 + 1 = 7,619,855   ← 7,654,321 is ≥ this
#48  →  47 × 165,649 + 1 = 7,785,504   ← and < this, so go to #47
```

**Internal page #47 entries** — entry *k* points to leaf `46 × 407 + k`, starting at:

```text
first id = 7,619,855 + (k − 1) × 407

#85  →  7,619,855 + 84 × 407 = 7,654,043  → leaf 46 × 407 + 85 = 18,807
#86  →  7,619,855 + 85 × 407 = 7,654,450  ← too big, so #85 is the match
```

**Leaf #18,807 entries** — entry *k* holds id `7,654,043 + (k − 1)`:

```text
7,654,321 − 7,654,043 + 1 = entry #279

Its pointer:  page = ceil(7,654,321 ÷ 70) = 109,348
              slot = 7,654,321 − 109,347 × 70 = 31
```

**Heap page #109,348** — holds ids `109,347 × 70 + 1 = 7,654,291` through `109,348 × 70 = 7,654,360`. Slot 31 is id 7,654,321.

## Why the sibling links matter

The `prev` / `next` pointers in the special space connect every page at the same level into a list. For a range query:

```sql
SELECT * FROM users WHERE id BETWEEN 7654300 AND 7655000;
```

the engine descends the tree once to leaf #18,807, then walks right through `next` to leaf #18,808, and stops when keys pass 7,655,000. It does not go back up to the root for each leaf.

---

## The search for id 7,654,321, page by page

### Level 2 — Root page (61 entries)

Entry #47 matches, because 7,619,855 ≤ 7,654,321 < 7,785,504 (entry #48's key).

```text
  byte 0                                                                                                        byte 8,192
                              pd_lower=268 ▼                ▼ pd_upper=7,200
  ┌────────┬───────┬───┬───────┬───┬───────┬────────────────┬─────────────┬───┬─────────────┬───┬─────────────┬───────────┐
  │ Header │   LP  │   │   LP  │   │   LP  │   FREE SPACE   │  Entry #61  │   │  Entry #47  │   │   Entry #1  │  Special  │
  │  24 B  │   #1  │ … │  #47  │ … │  #61  │  → 6,932 B ←   │  9,938,941  │ … │  7,619,855  │ … │      1      │    16 B   │
  │        │  4 B  │   │  4 B  │   │  4 B  │                │   → int 61  │   │   → int 47  │   │   → int 1   │           │
  └────────┴───┬───┴───┴───┬───┴───┴───┬───┴────────────────┴──────┬──────┴───┴──────┬──────┴───┴──────┬──────┴───────────┘
               │           │           │                           ▲                 ▲                 ▲
               │           │           └───────────────────────────┘                 │                 │
               │           └─────────────────────────────────────────────────────────┘                 │
               └───────────────────────────────────────────────────────────────────────────────────────┘
```

> Free space = 8,192 − 24 − (61 × 4) − (61 × 16) − 16 = 6,932 bytes. The root is mostly empty.

**↓ Follow entry #47 to internal page 47**

### Level 1 — Internal page #47 (407 entries)

Entry #85 matches, because 7,654,043 ≤ 7,654,321 < 7,654,450 (entry #86's key).

```text
  byte 0                                                                                                        byte 8,192
                            pd_lower=1,652 ▼                ▼ pd_upper=1,664
  ┌────────┬───────┬───┬───────┬───┬───────┬────────────────┬─────────────┬───┬─────────────┬───┬─────────────┬───────────┐
  │ Header │   LP  │   │   LP  │   │   LP  │   FREE SPACE   │  Entry #407 │   │  Entry #85  │   │   Entry #1  │  Special  │
  │  24 B  │   #1  │ … │  #85  │ … │  #407 │    → 12 B ←    │  7,785,097  │ … │  7,654,043  │ … │  7,619,855  │    16 B   │
  │        │  4 B  │   │  4 B  │   │  4 B  │                │→ leaf 19,129│   │→ leaf 18,807│   │→ leaf 18,723│           │
  └────────┴───┬───┴───┴───┬───┴───┴───┬───┴────────────────┴──────┬──────┴───┴──────┬──────┴───┴──────┬──────┴───────────┘
               │           │           │                           ▲                 ▲                 ▲
               │           │           └───────────────────────────┘                 │                 │
               │           └─────────────────────────────────────────────────────────┘                 │
               └───────────────────────────────────────────────────────────────────────────────────────┘
```

> Free space = 8,192 − 24 − (407 × 4) − (407 × 16) − 16 = 12 bytes. The page is full.

**↓ Follow entry #85 to leaf page 18,807**

### Level 0 — Leaf page #18,807 (407 entries)

Entry #279 holds the exact key 7,654,321 and the row's address (page 109,348, slot 31).

```text
  byte 0                                                                                                        byte 8,192
                            pd_lower=1,652 ▼                ▼ pd_upper=1,664
  ┌────────┬───────┬───┬───────┬───┬───────┬────────────────┬─────────────┬───┬─────────────┬───┬─────────────┬───────────┐
  │ Header │   LP  │   │   LP  │   │   LP  │   FREE SPACE   │  Entry #407 │   │  Entry #279 │   │   Entry #1  │  Special  │
  │  24 B  │   #1  │ … │  #279 │ … │  #407 │    → 12 B ←    │  7,654,449  │ … │  7,654,321  │ … │  7,654,043  │    16 B   │
  │        │  4 B  │   │  4 B  │   │  4 B  │                │→ (109350,19)│   │→ (109348,31)│   │→ (109344,33)│           │
  └────────┴───┬───┴───┴───┬───┴───┴───┬───┴────────────────┴──────┬──────┴───┴──────┬──────┴───┴──────┬──────┴───────────┘
               │           │           │                           ▲                 ▲                 ▲
               │           │           └───────────────────────────┘                 │                 │
               │           └─────────────────────────────────────────────────────────┘                 │
               └───────────────────────────────────────────────────────────────────────────────────────┘
```

> Same byte budget as the internal page: 12 bytes free.

**↓ Follow (109,348, 31) to the table page**

### Table (heap) page #109,348 (70 rows)

Line pointer #31 gives the byte offset of the row for id 7,654,321.

```text
  byte 0                                                                                            byte 8,192
                              pd_lower=304 ▼                ▼ pd_upper=352
  ┌────────┬───────┬───┬───────┬───┬───────┬────────────────┬─────────────┬───┬─────────────┬───┬─────────────┐
  │ Header │   LP  │   │   LP  │   │   LP  │   FREE SPACE   │ Row slot 70 │   │ Row slot 31 │   │  Row slot 1 │
  │  24 B  │   #1  │ … │  #31  │ … │  #70  │    → 48 B ←    │  7,654,360  │ … │  7,654,321  │ … │  7,654,291  │
  │        │  4 B  │   │  4 B  │   │  4 B  │                │  Sneha Das  │   │  Neha Gupta │   │ Ishaan Verma│
  └────────┴───┬───┴───┴───┬───┴───┴───┬───┴────────────────┴──────┬──────┴───┴──────┬──────┴───┴──────┬──────┘
               │           │           │                           ▲                 ▲                 ▲
               │           │           └───────────────────────────┘                 │                 │
               │           └─────────────────────────────────────────────────────────┘                 │
               └───────────────────────────────────────────────────────────────────────────────────────┘
```

> Free space = 8,192 − 24 − (70 × 4) − (70 × 112) = 48 bytes. No special space on table pages.

## Byte budget summary

| Page               | Header | Line pointers    | Free  | Entries / rows   | Special | Total     |
|--------------------|--------|------------------|-------|------------------|---------|-----------|
| Root (level 2)     | 24     | 61 × 4 = 244     | 6,932 | 61 × 16 = 976    | 16      | **8,192** |
| Internal #47 (1)   | 24     | 407 × 4 = 1,628  | 12    | 407 × 16 = 6,512 | 16      | **8,192** |
| Leaf #18,807 (0)   | 24     | 407 × 4 = 1,628  | 12    | 407 × 16 = 6,512 | 16      | **8,192** |
| Heap #109,348      | 24     | 70 × 4 = 280     | 48    | 70 × 112 = 7,840 | 0       | **8,192** |

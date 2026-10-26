# B-tree pages in slotted-page layout

## Why is the root page "level 2"?

B-tree levels are counted **from the bottom up**: leaf pages are level 0, and each level above adds 1. Our tree has 3 levels, so the root is level 3 − 1 = 2.

```text
Level 2 (root)       [ Root ]                                            1 page
                    /    |    \
Level 1         [Int 1] … [Int 47] … [Int 61]                          61 pages
                          /    |    \
Level 0 (leaf)  [Leaf 18,723] … [Leaf 18,807] … [Leaf 19,129]      24,571 pages
```

Counting from the bottom is used because leaves stay at level 0 forever. When the root splits and a new root is added on top, only the new root gets a new number; no existing page has to be renumbered.

## Why does the root have 61 line pointers?

A page has **one line pointer per entry**, and the root has one entry per child page. So the question is how many internal pages exist:

```text
Leaf pages     = 10,000,000 ÷ 407 = 24,570.02 → 24,571
Internal pages = 24,571 ÷ 407     = 60.37     → 61
```

61 internal pages → 61 entries in the root → **61 line pointers**. The same rule gives 407 line pointers in a full internal or leaf page, and 70 in a table page holding 70 rows.

## How to read the page diagrams

```text
byte 0                                                                  byte 8,191
┌────────┬───────────────────┬──────────────┬─────────────────────┬─────────┐
│ Header │ Line pointers  ──►│  FREE SPACE  │◄──  Entries / rows  │ Special │
└────────┴───────────────────┴──────────────┴─────────────────────┴─────────┘
           grow left → right   shrinks from    grow right → left
                               both sides
```

- **Header (24 B):** page metadata, including `pd_lower` (where free space starts) and `pd_upper` (where it ends).
- **Line pointers (4 B each):** stored in key order; each one holds the byte offset of its entry.
- **Free space:** the gap between `pd_lower` and `pd_upper`.
- **Entries / rows:** added from the end of the page backward, so entry #1 sits nearest the end.
- **Special (16 B):** B-tree pages only; holds the level and the previous/next sibling page numbers. Table (heap) pages don't have it.

The lines under each page connect a line pointer to the entry it points at. The diagrams are not to scale, and `…` hides the entries in between.

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

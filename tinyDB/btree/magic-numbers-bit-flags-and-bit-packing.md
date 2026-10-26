# Magic Numbers, Bit Flags and Bit-Packing in a Storage Engine

### A step-by-step guide using the `users` index from our running example

---

## What this guide covers

A storage engine has to squeeze a lot of information into very few bytes, and it has to protect itself from reading the wrong data. Three small techniques do most of that work:

| Technique | Question it answers | Where it lives in our example |
|-----------|---------------------|-------------------------------|
| **Magic number** | "Is this file really what I think it is?" | First 4 bytes of the index metapage (page 0) |
| **Bit flags** | "Which yes/no properties does this page or cell have?" | The flags byte at the start of every cell and page |
| **Bit-packing** | "How do I fit several small numbers into one integer?" | The 4-byte slot (line pointer) that points to each cell |

The running example is the same as before: a `users` table with 10 million rows, a B-tree index file `users_pkey.idx` whose root is page 290 at level 2, leaf page #18,807, and the cell for `id = 7,654,321` at byte offset 8,153, 23 bytes long.

All output shown in this guide comes from running `MagicAndFlags.java` (full code in section 10).

---

# Part 1: Magic numbers

## Step 1: What is a magic number?

A **magic number** is a fixed, unusual value written at a known position in a file, usually the very first bytes. A program that opens the file reads those bytes first. If they match the expected value, it continues. If not, it stops before misreading anything.

You have already seen one in this project. When we traced the Java demo with `strace`, the JVM's own file reads started with these bytes:

```text
pread64(3, "\312\376\272\276 ...", ...)
            CA   FE   BA   BE   →  0xCAFEBABE
```

`0xCAFEBABE` is the magic number at the start of every Java `.class` file. The JVM checks it before loading any class.

## Step 2: Why a storage engine needs one

Remember how the database finds the root of an index: it reads **page 0** and trusts the number it finds there. In our example that says "root = page 290". If page 0 contained garbage, the engine would jump to a random page and interpret random bytes as keys and child pointers.

A magic number protects against four real problems:

| Problem | What happens without a check | What the magic number catches |
|---------|------------------------------|-------------------------------|
| **Wrong file** | A config or text file is opened as an index, so its bytes are parsed as page numbers | The first bytes don't match |
| **Corruption** | Page 0 was overwritten by a bad write or disk error | The first bytes don't match |
| **Wrong format version** | An old engine opens a file written in a newer layout | A version number stored next to the magic doesn't match |
| **Wrong byte order** | A file written on a big-endian machine is read on a little-endian one, so every number comes out scrambled | The magic appears with its bytes reversed |

The check costs one integer comparison, done once when the file is opened.

## Step 3: Real-world magic numbers

| Format | Magic number | Position |
|--------|--------------|----------|
| **PostgreSQL B-tree index** | `0x053162` (`BTREE_MAGIC`), followed by a version number (currently 4) | Start of the metapage, block 0 of every B-tree index file |
| **SQLite database** | The 16 bytes `"SQLite format 3\0"` | Start of the database file |
| **Java class file** | `CA FE BA BE` | Start of the file |
| **Linux executable (ELF)** | `7F 45 4C 46` (`\x7F` then `"ELF"`) | Start of the file |
| **ZIP, JAR, DOCX, XLSX** | `50 4B 03 04` (`"PK"`, the creator's initials) | Start of the file |
| **PNG image** | `89 50 4E 47 0D 0A 1A 0A` | Start of the file |
| **Apache Parquet** | `"PAR1"` | Start **and** end of the file |

Notice the PostgreSQL entry: it is the exact technique our example uses. Every B-tree index in PostgreSQL begins with `0x053162` and a version number, before the root page number.

## Step 4: Design the metapage for `users_pkey.idx`

We put the magic number first, then a format version, then the fields the engine actually needs:

```text
users_pkey.idx — page 0 (metapage)

 byte   0           4           8           12          16                       8191
      ┌───────────┬───────────┬───────────┬───────────┬──────────── … ──────────────┐
      │   magic   │  version  │ root page │ root level│      rest of the page       │
      │ 0x053162  │     4     │    290    │     2     │                             │
 hex: │00 05 31 62│00 00 00 04│00 00 01 22│00 00 00 02│                             │
      └───────────┴───────────┴───────────┴───────────┴──────────── … ──────────────┘
```

Order matters:

1. **Magic first**, because it must be checkable before the reader knows anything else about the file.
2. **Version second**, so the format can change in future without old engines misreading new files.
3. **Real data after**, which is only trusted once both checks pass.

## Step 5: Write and validate the magic number in Java

**Writing** the metapage:

```java
static final int BTREE_MAGIC = 0x053162;
static final int BTREE_VERSION = 4;

ByteBuffer page = ByteBuffer.allocate(PAGE_SIZE);
page.putInt(BTREE_MAGIC)     // bytes 0..3
    .putInt(BTREE_VERSION)   // bytes 4..7
    .putInt(290)             // bytes 8..11: root page
    .putInt(2);              // bytes 12..15: root level
```

**Reading** it back, checking before trusting anything:

```java
int magic = page.getInt(0);

if (magic == BTREE_MAGIC) {
    int version = page.getInt(4);
    if (version != BTREE_VERSION) reject("unsupported version");
    int rootPage = page.getInt(8);        // only now is this trusted
} else if (magic == Integer.reverseBytes(BTREE_MAGIC)) {
    reject("written with the opposite byte order");
} else {
    reject("not a B-tree index, or page 0 is corrupted");
}
```

`Integer.reverseBytes(0x00053162)` is `0x62310500`. If the file shows that value, the data itself is probably fine, but it was written with the opposite byte order.

## Step 6: Test it against four files

The demo writes four files and opens each one:

```text
Opening users_pkey.idx
  first 4 bytes = 0x00053162
  OK: B-tree index, version 4, root = page 290, root level = 2

Opening notes.txt
  first 4 bytes = 0x68656C6C
  REJECT: not a B-tree index file (or page 0 is corrupted)

Opening from_other_machine.idx
  first 4 bytes = 0x62310500
  REJECT: magic is byte-swapped -> file was written on a machine with the opposite byte order

Opening corrupted.idx
  first 4 bytes = 0x00000000
  REJECT: not a B-tree index file (or page 0 is corrupted)
```

| File | First 4 bytes | Why |
|------|---------------|-----|
| `users_pkey.idx` | `00 05 31 62` | The real magic: accepted |
| `notes.txt` | `68 65 6C 6C` | The ASCII codes of `"hell"` from `"hello, …"` |
| `from_other_machine.idx` | `62 31 05 00` | The same magic, written little-endian |
| `corrupted.idx` | `00 00 00 00` | Page 0 was overwritten with zeros |

Without the check, `notes.txt` would have been read as "root = page 1,751,741,216" (bytes 8–11 are the text `"his "`, which is `0x68697320`), and the engine would try to read a page about 14 TB into a 32-byte file.

## Step 7: What makes a good magic number

- **Unlikely to occur by accident.** Avoid `0`, `1`, `0xFFFFFFFF` or plain text. Zeros are exactly what a blank or wiped page contains.
- **Not symmetric.** `0x053162` reversed is `0x62310500`, so a byte-order mistake is detectable. A value like `0xABBAABBA` would look the same in both orders.
- **Paired with a version number**, so one magic covers a whole family of formats.
- **Checked once, early.** A magic number only verifies the file header.

> A magic number does **not** verify the rest of the file. A single flipped bit inside leaf page #18,807 wouldn't change page 0's magic at all. For that, databases add a **checksum** to each page: PostgreSQL can store a 16-bit checksum in every page header and verify it on each read.

---

# Part 2: Bits and bit flags

## Step 8: Bits and hex, a quick refresher

One byte is 8 bits. Each bit is numbered from the right, starting at 0, and has a value of 2 to the power of its position:

```text
bit position:    7    6    5    4    3    2    1    0
bit value:     128   64   32   16    8    4    2    1
```

One hex digit is exactly 4 bits, which is why masks are usually written in hex:

| Hex | Binary | Decimal | Meaning as a mask |
|-----|--------|---------|-------------------|
| `0x01` | `0000 0001` | 1 | bit 0 |
| `0x02` | `0000 0010` | 2 | bit 1 |
| `0x04` | `0000 0100` | 4 | bit 2 |
| `0x08` | `0000 1000` | 8 | bit 3 |
| `0x05` | `0000 0101` | 5 | bits 0 and 2 |
| `0xFF` | `1111 1111` | 255 | all 8 bits |

`1 << n` means "1 shifted left by n places", which gives a number with only bit n set:

```text
1 << 0 = 0000 0001 = 0x01
1 << 1 = 0000 0010 = 0x02
1 << 2 = 0000 0100 = 0x04
```

So `HAS_OVERFLOW_PAGES = 0x04` and `(1 << 2)` are the same value, written two ways.

## Step 9: Why store flags as bits?

Leaf page #18,807 needs to record three yes/no facts:

- Is it a leaf page?
- Do its cells have variable-size values?
- Do some of its cells continue on overflow pages?

| Approach | Space | Notes |
|----------|-------|-------|
| One `boolean` field each | 3 bytes (Java stores a boolean array element as 1 byte) | 7 of every 8 bits wasted |
| One bit each, in one byte | **1 byte**, with room for 5 more flags | Fits in the fixed-size header of a page or cell |

The saving looks tiny until it is multiplied: a flags byte appears in every cell, and our table has 10 million rows. Saving even 2 bytes per row is 20 MB, which is about 2,400 fewer 8 KB pages to read, write and cache.

Bit flags also make some checks faster. Testing "is this a leaf with overflow pages?" is one AND and one compare, however many flags are involved.

## Step 10: Define the masks

A **mask** is a number with 1s only in the bit positions you care about. Each flag gets its own bit:

```java
static final int IS_LEAF_MASK         = 0x01;  // bit 0: 0000 0001
static final int VARIABLE_SIZE_VALUES = 0x02;  // bit 1: 0000 0010
static final int HAS_OVERFLOW_PAGES   = 0x04;  // bit 2: 0000 0100
```

Each mask is a power of 2, so no two masks share a bit, and any combination of flags gives a unique value.

## Step 11: Corrections to the snippet in the question

The snippet in the question has three bugs that are very common, so they are worth looking at closely.

**Bug 1: `0x01h` is not valid.**
`0x` (C, Java) and a trailing `h` (some assemblers) are two different ways of saying "hex". Use one: in Java, `0x01`.

**Bug 2: `&&` is not `&`.**

| Operator | Name | Works on | Result |
|----------|------|----------|--------|
| `&` | bitwise AND | each pair of bits | a number |
| `&&` | logical AND | two true/false values | true or false |

In Java, `flags && HAS_OVERFLOW_PAGES` does not compile, because `&&` needs booleans. In C it compiles, and returns 1 whenever **both values are non-zero**. That makes it a silent bug:

```text
flags = 0000 0001   (leaf only, no overflow)

C:     flags && 0x04   →  1 && 1 (both non-zero)  →  1   ✗ says overflow is set
Right: flags &  0x04   →  0000 0001 & 0000 0100   →  0   ✓ overflow is not set
```

**Bug 3: the last line overwrites the flags.**

```c
flags &= (flags && (1 << 2)) != 0;     // wrong
```

This mixes a test with an assignment. In C, the right side evaluates to `1` or `0`, so the line becomes `flags &= 1` or `flags &= 0`. If `flags` was `0000 0101`, it becomes `0000 0001`: the overflow flag is wiped. Or, if the test is false, everything is wiped.

**Corrected version:**

```java
static final int IS_LEAF_MASK         = 0x01;
static final int VARIABLE_SIZE_VALUES = 0x02;
static final int HAS_OVERFLOW_PAGES   = 0x04;

// Set the bit
flags |= HAS_OVERFLOW_PAGES;
flags |= (1 << 2);                                  // same thing

// Unset (clear) the bit
flags &= ~HAS_OVERFLOW_PAGES;
flags &= ~(1 << 2);                                 // same thing

// Test whether the bit is set (store the answer separately)
boolean isSet = (flags & HAS_OVERFLOW_PAGES) != 0;
boolean isSet2 = (flags & (1 << 2)) != 0;           // same thing
```

## Step 12: The four bitwise operators

Each one works on every bit position independently.

```text
 a  b │  a & b  │  a | b  │  a ^ b       a │ ~a
──────┼─────────┼─────────┼────────    ────┼────
 0  0 │    0    │    0    │    0         0 │  1
 0  1 │    0    │    1    │    1         1 │  0
 1  0 │    0    │    1    │    1
 1  1 │    1    │    1    │    0
```

The useful way to remember them, in terms of masks:

- **`x | 1` is always 1**, and **`x | 0` is `x`**. So OR with a mask *forces* the mask's bits on and leaves the rest alone.
- **`x & 0` is always 0**, and **`x & 1` is `x`**. So AND with a mask *keeps* the mask's bits and clears the rest.
- **`x ^ 1` is the opposite of `x`**, and **`x ^ 0` is `x`**. So XOR with a mask *flips* the mask's bits.
- **`~`** turns every 0 into 1 and every 1 into 0.

## Step 13: Set a bit with OR

Start with an empty flags byte, and mark leaf page #18,807 as a leaf, then as having overflow pages:

```text
Set IS_LEAF:                       Set HAS_OVERFLOW_PAGES:

  flags      0000 0000               flags      0000 0001
| mask       0000 0001             | mask       0000 0100
  ─────────────────────              ─────────────────────
  result     0000 0001  = 0x01       result     0000 0101  = 0x05
                     ▲                                ▲  ▲
                bit 0 forced on                 bit 2 on, bit 0 kept
```

Setting a bit that is already set changes nothing (`1 | 1 = 1`), so it is safe to repeat:

```text
  set IS_LEAF:            flags |= 0x01        flags = 0000 0001  (0x01)
  set HAS_OVERFLOW_PAGES: flags |= 0x04        flags = 0000 0101  (0x05)
  set it again (no change)                     flags = 0000 0101  (0x05)
```

## Step 14: Test a bit with AND

AND the flags with a mask. Every bit outside the mask becomes 0, so the result is either 0 (the bit was off) or the mask itself (the bit was on):

```text
Is it a leaf?               Variable-size values?       Overflow pages?

  flags   0000 0101           flags   0000 0101           flags   0000 0101
& mask    0000 0001         & mask    0000 0010         & mask    0000 0100
  ───────────────────         ───────────────────         ───────────────────
  result  0000 0001           result  0000 0000           result  0000 0100
          ≠ 0 → true                  = 0 → false                 ≠ 0 → true
```

```text
  test: leaf=true, variableSize=false, overflow=true
```

Always compare the result with `!= 0`, not `== 1`. The result for the overflow flag is `0000 0100` = 4, not 1, so `== 1` would wrongly return false.

## Step 15: Clear a bit with AND NOT

To turn off just the overflow flag:

1. `~HAS_OVERFLOW_PAGES` builds a mask with **every bit on except bit 2**.
2. AND keeps every bit where that mask is 1, and clears bit 2.

```text
  HAS_OVERFLOW_PAGES     0000 0100
  ~HAS_OVERFLOW_PAGES    1111 1011     ← step 1: invert

  flags                  0000 0101
& ~HAS_OVERFLOW_PAGES    1111 1011     ← step 2: AND
  ─────────────────────────────────
  result                 0000 0001  = 0x01
                               ▲
                     bit 2 cleared, bit 0 kept
```

```text
  clear HAS_OVERFLOW_PAGES: flags &= ~0x04     flags = 0000 0001  (0x01)
```

Why not just subtract 4? Because `flags - 4` only works if the bit was set. If it wasn't, subtracting corrupts other bits. `&= ~mask` is always safe.

## Step 16: Toggle with XOR, and work with several flags at once

**Toggle** flips a bit whatever its current value:

```text
  toggle VARIABLE_SIZE:   flags ^= 0x02        flags = 0000 0011  (0x03)
  toggle again:           flags ^= 0x02        flags = 0000 0001  (0x01)
```

**Several flags at once:** combine masks with OR, then use the same operations.

```java
int both = IS_LEAF_MASK | HAS_OVERFLOW_PAGES;            // 0000 0101

flags |= both;                                           // set both
flags &= ~both;                                          // clear both
boolean all = (flags & both) == both;                    // are BOTH set?
boolean any = (flags & both) != 0;                       // is EITHER set?
```

Note the difference between the two tests: "all" compares with the mask, "any" compares with zero.

```text
  set two flags at once:  flags |= 0x05        flags = 0000 0101  (0x05)
  test ALL of 0x05 set: true,  test ANY of 0x06 set: true
```

## Step 17: Common pitfalls

**Operator precedence.** In both C and Java, `==` and `!=` bind more tightly than `&`:

```text
flags & MASK != 0          parsed as   flags & (MASK != 0)
(flags & MASK) != 0        what you meant
```

In C the first form compiles and gives a wrong answer. In Java it fails to compile. Always add the parentheses.

**Java's `byte` is signed.** If the flags are stored in a `byte`, bit 7 makes the value negative, and the byte is widened to an `int` before any bitwise operation. Mask with `0xFF` when reading:

```java
byte stored = (byte) 0x85;     // 1000 0101, which Java sees as -123
int flags = stored & 0xFF;     // 133, safe to test
```

**Masks must not overlap.** Using `0x03` as a single flag's mask, for example, would make it share bits with both `0x01` and `0x02`.

## Step 18: Real database flags

The same three flag values from the question appear in PostgreSQL's row header. Every heap row (every row cell, like Neha Gupta's from the previous article) has a 16-bit `t_infomask`:

| Bit | PostgreSQL name | Meaning |
|-----|-----------------|---------|
| `0x0001` | `HEAP_HASNULL` | The row has at least one NULL column (so a null bitmap follows) |
| `0x0002` | `HEAP_HASVARWIDTH` | The row has variable-width columns |
| `0x0004` | `HEAP_HASEXTERNAL` | Some value is stored out-of-line in TOAST, PostgreSQL's overflow storage |

For Neha's row: there are no NULLs, it has `VARCHAR` columns, and nothing is in TOAST, so these bits are `0000 0010`. If she had a long `bio` column that was moved to TOAST, the row would get `0x0004` set: exactly the `HAS_OVERFLOW_PAGES` idea.

PostgreSQL B-tree pages have their own flag field (`btpo_flags`) in the special space at the end of each page:

| Bit | Name | Meaning |
|-----|------|---------|
| `1 << 0` | `BTP_LEAF` | This is a leaf page |
| `1 << 1` | `BTP_ROOT` | This is the root page |
| `1 << 2` | `BTP_DELETED` | The page has been deleted from the tree |
| `1 << 3` | `BTP_META` | This is the metapage |

In our example, the root page 290 has `BTP_ROOT` set, and leaf page #18,807 has `BTP_LEAF` set.

---

# Part 3: Bit-packed data

## Step 19: From 1-bit flags to multi-bit fields

Flags use one bit per yes/no value. **Bit-packing** takes the same idea further and gives each *small number* just enough bits, then stores several of them in one integer.

The rule for "just enough": a field with *n* bits can hold values from 0 to 2ⁿ − 1.

| Bits | Max value | Enough for |
|------|-----------|------------|
| 2 | 3 | 4 states |
| 13 | 8,191 | sizes within one 8 KB page |
| 15 | 32,767 | any byte offset in a page up to 32 KB |

## Step 20: The 4-byte slot is bit-packed

In the earlier articles every slot (line pointer) was 4 bytes. It actually holds **three** values:

| Field | Bits | Range | Value for id 7,654,321's cell |
|-------|------|-------|-------------------------------|
| `lp_off`: where the cell starts | 15 | 0–32,767 | 8,153 |
| `lp_flags`: the slot's state | 2 | 0–3 (unused, normal, redirect, dead) | 1 (normal) |
| `lp_len`: the cell's length | 15 | 0–32,767 | 23 |
| **Total** | **32** | | |

Stored as three separate fields, this would take at least 2 + 1 + 2 = 5 bytes, or 12 bytes as three Java `int`s. Packed, it is 4 bytes. With 407 slots per page, that saves at least 407 bytes per page compared to the 5-byte version, which is room for 20 more entries.

This is exactly how PostgreSQL defines its line pointer (`ItemIdData`). PostgreSQL uses C bit-fields and lets the compiler place them; the layout below matches what common compilers produce on x86, with `lp_off` in the lowest bits.

```text
bit:  31                           17 16  15 14                            0
     ┌───────────────────────────────┬──────┬───────────────────────────────┐
     │          lp_len (15)          │flags │          lp_off (15)          │
     │                               │ (2)  │                               │
     └───────────────────────────────┴──────┴───────────────────────────────┘
       shift = 17                     shift   shift = 0
       mask  = 0x7FFF                 = 15    mask  = 0x7FFF
                                      mask
                                      = 0x3
```

## Step 21: Packing with shift and OR

Each field is masked to its width (so it can't spill into its neighbour), shifted left to its position, and then all three are ORed together:

```java
static final int OFF_SHIFT = 0,  FLAG_SHIFT = 15, LEN_SHIFT = 17;
static final int OFF_MASK = 0x7FFF, FLAG_MASK = 0x3, LEN_MASK = 0x7FFF;

static int packSlot(int off, int flags, int len) {
    return (off   & OFF_MASK)  << OFF_SHIFT
         | (flags & FLAG_MASK) << FLAG_SHIFT
         | (len   & LEN_MASK)  << LEN_SHIFT;
}
```

Step by step for offset 8,153, flags 1, length 23:

```text
                                   lp_len (15)     fl  lp_off (15)
off   = 8,153  << 0             →  000000000000000 00 001111111011001
flags = 1      << 15            →  000000000000000 01 000000000000000
len   = 23     << 17            →  000000000010111 00 000000000000000
                                   ─────────────────────────────────── OR
packed = 3,055,577 = 0x002E9FD9 →  000000000010111 01 001111111011001
```

Because each field occupies different bit positions, OR combines them without any of them interfering with the others.

## Step 22: Unpacking with shift and AND

Reverse the process: shift the field down to bit 0, then AND with its mask to drop everything else:

```java
static int slotOff(int slot)   { return (slot >>> 0)  & 0x7FFF; }
static int slotFlags(int slot) { return (slot >>> 15) & 0x3; }
static int slotLen(int slot)   { return (slot >>> 17) & 0x7FFF; }
```

For the flags field:

```text
slot          00000000001011101001111111011001
slot >>> 15   00000000000000000000000001011101     ← lp_off shifted out; flags now in bits 0–1
& 0x3         00000000000000000000000000000011     ← keep only the lowest 2 bits
              ────────────────────────────────
result        00000000000000000000000000000001  =  1 (normal)
```

Java's `>>>` shifts in zeros from the left. The plain `>>` copies the sign bit instead, which gives wrong results whenever bit 31 is set, so `>>>` is the safe choice for unpacking.

## Step 23: Changing one field in place

When the row with id 7,654,321 is deleted and vacuumed, its slot is marked **dead**. Only the 2 flag bits should change; the offset and length bits must stay exactly the same. It's the same clear-then-set pattern used for single flags:

```java
slot = (slot & ~(FLAG_MASK << FLAG_SHIFT))   // 1. clear bits 15–16
     | (LP_DEAD << FLAG_SHIFT);              // 2. put 3 (dead) there
```

```text
FLAG_MASK << 15              000000000000000 11 000000000000000
~(FLAG_MASK << 15)           111111111111111 00 111111111111111
slot & ~(…)                  000000000010111 00 001111111011001   flags cleared
| (3 << 15)                  000000000000000 11 000000000000000
                             ─────────────────────────────────────
new slot = 0x002F9FD9        000000000010111 11 001111111011001   off and len unchanged
```

Demo output:

```text
Line pointer for the cell of id 7,654,321 (offset 8153, length 23)
  packed  = 3,055,577  (0x002E9FD9)
  bits    = 000000000010111 01 001111111011001
            len=23          fl  off=8153
  unpack  -> off=8153, flags=1, len=23
  mark dead -> 0x002F9FD9  off=8153, flags=3, len=23
```

## Step 24: Flags and a number in the same field

Bit-packing can mix a number and flags in one integer. PostgreSQL's index entry, the 16-byte entry from our index, starts with a 2-byte field called `t_info`:

```text
bit:  15     14     13     12                                     0
     ┌──────┬──────┬──────┬────────────────────────────────────────┐
     │ has  │ has  │ (AM  │            entry size (13 bits)        │
     │ nulls│ var- │ use) │            mask 0x1FFF, max 8,191      │
     │0x8000│width │      │                                        │
     │      │0x4000│      │                                        │
     └──────┴──────┴──────┴────────────────────────────────────────┘
```

Reading it uses the techniques from both parts of this guide:

```java
int size     = tInfo & 0x1FFF;            // multi-bit field: mask, no shift needed
boolean var  = (tInfo & 0x4000) != 0;     // single-bit flag
boolean null = (tInfo & 0x8000) != 0;     // single-bit flag
```

```text
Index entry t_info for id 7,654,321 (16-byte entry, BIGINT key)
  t_info = 0x0010 -> size=16, varwidth=false, nulls=false
  email index entry: t_info = 0x4020 -> size=32, varwidth=true, nulls=false
```

For our `id` index, the entry is 16 bytes with a fixed-width key and no NULLs, so `t_info` is just `0x0010`. An index on `email` would have variable-width keys, so `0x4000` is set as well.

This is also why the previous article said PostgreSQL's index entry doesn't need separate `key_size` and flags fields: both are squeezed into these 2 bytes.

---

# Part 4: Putting it together

## Step 25: Where each technique appears in our example

```text
users_pkey.idx

 page 0 (metapage)
┌───────────┬─────────┬───────────┬──────┐
│ 0x053162  │ ver 4   │ root=290  │ lvl 2│  ← MAGIC NUMBER checked first, then root trusted
└───────────┴─────────┴───────────┴──────┘
                        │
                        ▼
 page 18,807 (leaf)
┌────────┬────────────────────┬─────────┬──────────────────────────┬────────────────┐
│ header │ slots              │  free   │ cells                    │ special        │
│        │ 0x002E9FD9 …       │         │ [flags|sizes|key|value]  │ btpo_flags =   │
│        │ ▲ BIT-PACKED:      │         │  ▲ BIT FLAGS in the      │ BTP_LEAF       │
│        │   off|flags|len    │         │    cell's first byte     │ ▲ BIT FLAGS    │
└────────┴────────────────────┴─────────┴──────────────────────────┴────────────────┘
```

| Technique | Operation | Code |
|-----------|-----------|------|
| Magic number | check | `page.getInt(0) == BTREE_MAGIC` |
| Flag | set | `flags \|= MASK` |
| Flag | clear | `flags &= ~MASK` |
| Flag | test | `(flags & MASK) != 0` |
| Flag | toggle | `flags ^= MASK` |
| Packed field | pack | `(value & FIELD_MASK) << SHIFT`, OR the fields together |
| Packed field | unpack | `(packed >>> SHIFT) & FIELD_MASK` |
| Packed field | update | `(packed & ~(FIELD_MASK << SHIFT)) \| (value << SHIFT)` |

---

## Step 26: Full code

Save as `MagicAndFlags.java` and run with `java MagicAndFlags.java` (Java 11 or newer).

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.channels.FileChannel;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;

/**
 * Magic numbers, bit flags and bit-packed fields, using the users_pkey index
 * from our running example (10 million users, root page 290, 3 levels).
 *
 * Run with: java MagicAndFlags.java
 */
public class MagicAndFlags {

    // =====================================================================
    // PART 1: MAGIC NUMBERS
    // =====================================================================

    static final int PAGE_SIZE = 8192;

    /** Same value PostgreSQL stores in every B-tree metapage (BTREE_MAGIC). */
    static final int BTREE_MAGIC = 0x053162;
    static final int BTREE_VERSION = 4;

    /** Metapage layout: [magic:4][version:4][rootPage:4][rootLevel:4] */
    static void writeMetapage(Path path, ByteOrder order) throws IOException {
        ByteBuffer page = ByteBuffer.allocate(PAGE_SIZE).order(order);
        page.putInt(BTREE_MAGIC).putInt(BTREE_VERSION).putInt(290).putInt(2);
        page.rewind();
        try (FileChannel ch = FileChannel.open(path,
                StandardOpenOption.CREATE, StandardOpenOption.WRITE,
                StandardOpenOption.TRUNCATE_EXISTING)) {
            ch.write(page, 0);
        }
    }

    /** Read page 0 and check the magic number before trusting anything else in the file. */
    static void openIndex(Path path) throws IOException {
        System.out.println("Opening " + path.getFileName());
        if (Files.size(path) < 16) {
            System.out.println("  REJECT: file is smaller than a metapage header\n");
            return;
        }
        ByteBuffer page = ByteBuffer.allocate(16);  // big-endian by default
        try (FileChannel ch = FileChannel.open(path, StandardOpenOption.READ)) {
            ch.read(page, 0);
        }
        page.flip();
        int magic = page.getInt(0);
        System.out.printf("  first 4 bytes = 0x%08X%n", magic);

        if (magic == BTREE_MAGIC) {
            int version = page.getInt(4);
            if (version != BTREE_VERSION) {
                System.out.println("  REJECT: B-tree format version " + version
                        + ", this engine reads version " + BTREE_VERSION + "\n");
                return;
            }
            System.out.printf("  OK: B-tree index, version %d, root = page %d, root level = %d%n%n",
                    version, page.getInt(8), page.getInt(12));
        } else if (magic == Integer.reverseBytes(BTREE_MAGIC)) {
            System.out.println("  REJECT: magic is byte-swapped -> file was written on a machine"
                    + " with the opposite byte order\n");
        } else {
            System.out.println("  REJECT: not a B-tree index file (or page 0 is corrupted)\n");
        }
    }

    // =====================================================================
    // PART 2: SINGLE-BIT FLAGS
    // =====================================================================

    static final int IS_LEAF_MASK         = 0x01;  // bit 0: 0000 0001
    static final int VARIABLE_SIZE_VALUES = 0x02;  // bit 1: 0000 0010
    static final int HAS_OVERFLOW_PAGES   = 0x04;  // bit 2: 0000 0100

    static String bits8(int v) {
        String s = String.format("%8s", Integer.toBinaryString(v & 0xFF)).replace(' ', '0');
        return s.substring(0, 4) + " " + s.substring(4);
    }

    static void show(String step, int flags) {
        System.out.printf("  %-44s flags = %s  (0x%02X)%n", step, bits8(flags), flags);
    }

    static void flagDemo() {
        System.out.println("Flags byte of leaf page #18,807");
        int flags = 0;
        show("start", flags);

        flags |= IS_LEAF_MASK;
        show("set IS_LEAF:            flags |= 0x01", flags);

        flags |= HAS_OVERFLOW_PAGES;
        show("set HAS_OVERFLOW_PAGES: flags |= 0x04", flags);

        flags |= HAS_OVERFLOW_PAGES;
        show("set it again (no change)", flags);

        boolean isLeaf = (flags & IS_LEAF_MASK) != 0;
        boolean varSize = (flags & VARIABLE_SIZE_VALUES) != 0;
        boolean overflow = (flags & (1 << 2)) != 0;
        System.out.printf("  test: leaf=%b, variableSize=%b, overflow=%b%n", isLeaf, varSize, overflow);

        flags &= ~HAS_OVERFLOW_PAGES;
        show("clear HAS_OVERFLOW_PAGES: flags &= ~0x04", flags);

        flags ^= VARIABLE_SIZE_VALUES;
        show("toggle VARIABLE_SIZE:   flags ^= 0x02", flags);
        flags ^= VARIABLE_SIZE_VALUES;
        show("toggle again:           flags ^= 0x02", flags);

        int both = IS_LEAF_MASK | HAS_OVERFLOW_PAGES;
        flags |= both;
        show("set two flags at once:  flags |= 0x05", flags);
        System.out.printf("  test ALL of 0x05 set: %b,  test ANY of 0x06 set: %b%n%n",
                (flags & both) == both, (flags & (VARIABLE_SIZE_VALUES | HAS_OVERFLOW_PAGES)) != 0);
    }

    // =====================================================================
    // PART 3: BIT-PACKED MULTI-BIT FIELDS
    // =====================================================================

    /**
     * A 4-byte line pointer (slot), packed like PostgreSQL's ItemIdData:
     *   bits  0..14  lp_off   (15 bits) byte offset of the cell
     *   bits 15..16  lp_flags ( 2 bits) 0 unused, 1 normal, 2 redirect, 3 dead
     *   bits 17..31  lp_len   (15 bits) length of the cell in bytes
     */
    static final int OFF_BITS = 15, FLAG_BITS = 2, LEN_BITS = 15;
    static final int OFF_SHIFT = 0, FLAG_SHIFT = 15, LEN_SHIFT = 17;
    static final int OFF_MASK = (1 << OFF_BITS) - 1;    // 0x7FFF
    static final int FLAG_MASK = (1 << FLAG_BITS) - 1;  // 0x3
    static final int LEN_MASK = (1 << LEN_BITS) - 1;    // 0x7FFF
    static final int LP_NORMAL = 1, LP_DEAD = 3;

    static int packSlot(int off, int flags, int len) {
        return (off & OFF_MASK) << OFF_SHIFT
             | (flags & FLAG_MASK) << FLAG_SHIFT
             | (len & LEN_MASK) << LEN_SHIFT;
    }
    static int slotOff(int slot)   { return (slot >>> OFF_SHIFT) & OFF_MASK; }
    static int slotFlags(int slot) { return (slot >>> FLAG_SHIFT) & FLAG_MASK; }
    static int slotLen(int slot)   { return (slot >>> LEN_SHIFT) & LEN_MASK; }

    static String bits32(int v) {
        String s = String.format("%32s", Integer.toBinaryString(v)).replace(' ', '0');
        // group as len(15) | flags(2) | off(15)
        return s.substring(0, 15) + " " + s.substring(15, 17) + " " + s.substring(17);
    }

    static void slotDemo() {
        System.out.println("Line pointer for the cell of id 7,654,321 (offset 8153, length 23)");
        int slot = packSlot(8153, LP_NORMAL, 23);
        System.out.printf("  packed  = %,d  (0x%08X)%n", slot, slot);
        System.out.println("  bits    = " + bits32(slot));
        System.out.println("            len=23          fl  off=8153");
        System.out.printf("  unpack  -> off=%d, flags=%d, len=%d%n",
                slotOff(slot), slotFlags(slot), slotLen(slot));

        // Mark the row dead: replace only the 2 flag bits, keep the other 30 bits.
        slot = (slot & ~(FLAG_MASK << FLAG_SHIFT)) | (LP_DEAD << FLAG_SHIFT);
        System.out.printf("  mark dead -> 0x%08X  off=%d, flags=%d, len=%d%n%n",
                slot, slotOff(slot), slotFlags(slot), slotLen(slot));
    }

    /**
     * PostgreSQL IndexTuple t_info (2 bytes):
     *   bits 0..12  size of the index entry (13 bits, max 8191)
     *   bit  14     has variable-width key  (0x4000)
     *   bit  15     has nulls               (0x8000)
     */
    static final int INDEX_SIZE_MASK = 0x1FFF;
    static final int INDEX_VAR_MASK = 0x4000;
    static final int INDEX_NULL_MASK = 0x8000;

    static void tInfoDemo() {
        System.out.println("Index entry t_info for id 7,654,321 (16-byte entry, BIGINT key)");
        int tInfo = 16;  // size only: fixed-width key, no nulls
        System.out.printf("  t_info = 0x%04X -> size=%d, varwidth=%b, nulls=%b%n",
                tInfo, tInfo & INDEX_SIZE_MASK,
                (tInfo & INDEX_VAR_MASK) != 0, (tInfo & INDEX_NULL_MASK) != 0);

        int emailEntry = 32 | INDEX_VAR_MASK;  // an index on email: variable-width key
        System.out.printf("  email index entry: t_info = 0x%04X -> size=%d, varwidth=%b, nulls=%b%n",
                emailEntry, emailEntry & INDEX_SIZE_MASK,
                (emailEntry & INDEX_VAR_MASK) != 0, (emailEntry & INDEX_NULL_MASK) != 0);
    }

    // =====================================================================

    public static void main(String[] args) throws IOException {
        Path dir = Files.createTempDirectory("magic");

        System.out.println("=== PART 1: magic numbers ===\n");
        Path good = dir.resolve("users_pkey.idx");
        writeMetapage(good, ByteOrder.BIG_ENDIAN);
        openIndex(good);

        Path text = dir.resolve("notes.txt");
        Files.writeString(text, "hello, this is not an index file");
        openIndex(text);

        Path swapped = dir.resolve("from_other_machine.idx");
        writeMetapage(swapped, ByteOrder.LITTLE_ENDIAN);
        openIndex(swapped);

        Path corrupt = dir.resolve("corrupted.idx");
        writeMetapage(corrupt, ByteOrder.BIG_ENDIAN);
        try (FileChannel ch = FileChannel.open(corrupt, StandardOpenOption.WRITE)) {
            ch.write(ByteBuffer.wrap(new byte[]{0, 0, 0, 0}), 0);  // page 0 overwritten
        }
        openIndex(corrupt);

        System.out.println("=== PART 2: single-bit flags ===\n");
        flagDemo();

        System.out.println("=== PART 3: bit-packed fields ===\n");
        slotDemo();
        tInfoDemo();
    }
}
```

### Full output

```text
=== PART 1: magic numbers ===

Opening users_pkey.idx
  first 4 bytes = 0x00053162
  OK: B-tree index, version 4, root = page 290, root level = 2

Opening notes.txt
  first 4 bytes = 0x68656C6C
  REJECT: not a B-tree index file (or page 0 is corrupted)

Opening from_other_machine.idx
  first 4 bytes = 0x62310500
  REJECT: magic is byte-swapped -> file was written on a machine with the opposite byte order

Opening corrupted.idx
  first 4 bytes = 0x00000000
  REJECT: not a B-tree index file (or page 0 is corrupted)

=== PART 2: single-bit flags ===

Flags byte of leaf page #18,807
  start                                        flags = 0000 0000  (0x00)
  set IS_LEAF:            flags |= 0x01        flags = 0000 0001  (0x01)
  set HAS_OVERFLOW_PAGES: flags |= 0x04        flags = 0000 0101  (0x05)
  set it again (no change)                     flags = 0000 0101  (0x05)
  test: leaf=true, variableSize=false, overflow=true
  clear HAS_OVERFLOW_PAGES: flags &= ~0x04     flags = 0000 0001  (0x01)
  toggle VARIABLE_SIZE:   flags ^= 0x02        flags = 0000 0011  (0x03)
  toggle again:           flags ^= 0x02        flags = 0000 0001  (0x01)
  set two flags at once:  flags |= 0x05        flags = 0000 0101  (0x05)
  test ALL of 0x05 set: true,  test ANY of 0x06 set: true

=== PART 3: bit-packed fields ===

Line pointer for the cell of id 7,654,321 (offset 8153, length 23)
  packed  = 3,055,577  (0x002E9FD9)
  bits    = 000000000010111 01 001111111011001
            len=23          fl  off=8153
  unpack  -> off=8153, flags=1, len=23
  mark dead -> 0x002F9FD9  off=8153, flags=3, len=23

Index entry t_info for id 7,654,321 (16-byte entry, BIGINT key)
  t_info = 0x0010 -> size=16, varwidth=false, nulls=false
  email index entry: t_info = 0x4020 -> size=32, varwidth=true, nulls=false
```

---

## Summary

- A **magic number** is a fixed value at a known position that lets a program confirm a file's type, detect a wiped header and spot a byte-order mismatch, all before trusting anything else in the file. Pair it with a version number, and use page checksums to protect the rest of the file.
- **Bit flags** store yes/no facts one per bit. A **mask** selects the bits to work on: `|` sets them, `& ~` clears them, `&` tests them, `^` flips them.
- `&` and `&&` are different operators: `&` works on bits, `&&` works on true/false. Mixing them up is one of the most common bit-manipulation bugs.
- **Bit-packing** gives each small number just enough bits and stores several in one integer. Pack with shift and OR, unpack with shift and AND, update one field with clear-then-set.
- All three appear in real databases: PostgreSQL's `BTREE_MAGIC` in the index metapage, `t_infomask` and `btpo_flags` as bit flags, and the 4-byte line pointer and 2-byte `t_info` as bit-packed fields.

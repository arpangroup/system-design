# Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch

> **The interview question this guide answers:**
>
> *"Design a storage system. Along the way, be ready to explain block storage, file storage, and object storage — what each one actually is, not just their names — and how RAID protects data against disk failure. Then implement the core mechanics of each from scratch."*
>
> This guide is written as that interview would actually unfold: a main question, a requirements-gathering phase, a layered design, and a sequence of **follow-up questions** — each one escalating the difficulty exactly the way a real interviewer would, and each one answered by both an explanation and working code, not just a definition.

---

# 1. What We Are Building

We are building **three storage abstractions and one reliability layer**, each as a real, working implementation:

- **Block storage** — a raw, fixed-size-addressable storage device with no concept of files or names, the foundation everything else is built on.
- **File storage** — a filesystem layered on top of block storage: inodes, directories, path resolution, free-space tracking, and a journal for crash consistency.
- **Object storage** (S3-style) — a flat, key-addressed, immutable-object store with versioning and multipart upload, built independently of the filesystem hierarchy.
- **RAID** (0, 1, 5, 6, 10) — striping, mirroring, and parity-based reconstruction, implemented byte-for-byte, including the XOR math that lets one parity block recover an entire failed disk.

```text
Object Storage (§27-§37)          File Storage (§17-§26)
  flat key -> object                  path -> inode -> blocks
  immutable, versioned                hierarchical, mutable in place
        |                                     |
        +------------------+------------------+
                           v
                  Block Storage (§11-§16)
                  fixed-size addressable blocks, no names, no metadata
                           |
                           v
                  RAID (§38-§48)
                  striping / mirroring / parity across multiple physical disks
                           |
                           v
                    Physical Disks
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain the actual, mechanical difference between block, file, and object storage — not just recite that "S3 is object storage" — and justify which one fits a given workload.
- Implement an inode-based filesystem's core mechanics: block mapping, directories, path resolution, and free-space tracking.
- Explain why object storage doesn't support in-place mutation, and what versioning and multipart upload do instead.
- Derive XOR parity from first principles and explain exactly how RAID 5 reconstructs a failed disk's data from the survivors.
- Answer, with real justification, "which RAID level for which workload" — not from memorized rules of thumb, but from the actual read/write/failure-tolerance tradeoffs each level makes.
- Extend single-machine RAID reasoning into distributed erasure coding, and explain (at a conceptual level defensible in an interview) how S3 achieves its stated durability figures.

---

# 3. Why This Matters (The Interview, Framed)

Storage system design is a favorite senior/staff **infrastructure interview** topic precisely because it can't be answered from a single memorized diagram — a strong candidate has to reason across several distinct layers in the same conversation:

- **Abstraction boundaries** — knowing exactly what block storage does and doesn't provide, and why a filesystem or a database sits on top of it rather than replacing it.
- **Data structures under I/O constraints** — an inode table and a block bitmap are just data structures, but ones shaped entirely by "must survive being read from disk after a crash."
- **Failure reasoning** — RAID exists because disks fail; understanding *which* failure patterns each RAID level tolerates (and which it doesn't) is the actual signal an interviewer is probing for, not the acronym.
- **Scaling reasoning** — object storage's flat, immutable design isn't an arbitrary choice; it's what makes horizontal scaling across thousands of machines tractable in a way a POSIX-hierarchical filesystem is not.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | `RandomAccessFile`/`FileChannel` give precise, byte-level control over a backing "disk" file, exactly as needed to simulate block storage honestly. |
| Simulated disks | Plain local files, each treated as a raw block device | No real hardware or virtualization needed — a fixed-size file addressed by byte offset behaves identically to a block device for this guide's purposes. |
| Hashing (object storage keys, §30) | `java.security.MessageDigest` (SHA-256) | Standard library, no external dependency, matches how real content-addressable stores compute object identity. |
| No external storage libraries | — | The entire point is to build the mechanics real filesystems, RAID controllers, and S3-like systems are built from. |

---

# 5. Project Structure

```text
filestorage/
├── src/main/java/com/example/filestorage/
│   ├── block/
│   │   ├── BlockDevice.java              // §13
│   │   └── FileBackedBlockDevice.java
│   ├── filesystem/
│   │   ├── Inode.java                    // §18-§19
│   │   ├── SuperBlock.java
│   │   ├── BlockBitmap.java              // §23
│   │   ├── Directory.java                // §21
│   │   ├── MiniFileSystem.java           // §19-§22
│   │   └── Journal.java                  // §26
│   ├── objectstore/
│   │   ├── MiniObjectStore.java          // §29-§32
│   │   ├── ObjectVersion.java            // §34
│   │   └── MultipartUpload.java          // §36
│   └── raid/
│       ├── Raid0StripedDevice.java       // §41
│       ├── Raid1MirroredDevice.java      // §42
│       ├── Raid5Device.java              // §45
│       └── XorParity.java                // §44
└── src/test/java/com/example/filestorage/
    ├── MiniFileSystemTest.java
    ├── ObjectStoreVersioningTest.java
    └── Raid5ReconstructionTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

A candidate who starts drawing boxes the instant "design a storage system" is asked has already made a mistake — the question is deliberately underspecified, and the first job is narrowing it:

> **Candidate's clarifying questions:** "Is this for a single machine or a distributed system? Do consumers need a POSIX-style hierarchy (paths, directories), or is key-based access to opaque blobs enough? What's the durability requirement — can we tolerate losing data on a single disk failure? What's the read/write pattern — small frequent updates, or large immutable writes?"

These questions aren't stalling — they determine which of the three storage abstractions (§10) is even the right starting point, and the rest of this guide is structured to build **all three**, specifically so you can recognize which requirements point to which one, rather than defaulting to whichever you happen to know best.

---

# 7. Functional Requirements: What Must the System Do

For this guide's scope, settling on a concrete, interview-realistic requirement set:

- Store and retrieve data addressed by a raw block number (block storage, §11).
- Store and retrieve data addressed by a hierarchical path with named files and directories (file storage, §17).
- Store and retrieve data addressed by a flat key, immutable once written, with versioning (object storage, §27).
- Survive the failure of a single physical disk without data loss (RAID, §38).

---

# 8. Non-Functional Requirements: Durability, Availability, Latency, Throughput

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Durability** | Once a write is acknowledged, it survives a crash or disk failure | Journaling (§25-§26), RAID redundancy (§38-§48) |
| **Availability** | The system keeps serving reads/writes even while degraded (a disk has failed but hasn't been replaced yet) | RAID's degraded-mode read reconstruction (§45) |
| **Latency** | How long a single read/write takes | Block storage's O(1) addressed access (§13) vs. object storage's hash/index lookup (§30) |
| **Throughput** | How much data can move per second, especially under parallel access | RAID 0/10 striping across multiple disks (§39, §47) |

Naming these explicitly, and being honest that different storage types make different tradeoffs among them, is what separates a system-design answer from a trivia recitation.

---

# 9. Follow-up Question 1 — "What's the Difference Between Block, File, and Object Storage?"

> **Interviewer:** *"Before we design anything — in your own words, what's actually different between block storage, file storage, and object storage? Most people can name examples (EBS, NFS, S3) but not explain the mechanical difference."*

The honest, mechanical answer: they differ in **what unit of addressing the storage layer exposes to whoever is using it**, and **what metadata, if any, it manages on the caller's behalf.**

- **Block storage** exposes fixed-size numbered blocks. It has no idea what a "file" is — that's the caller's (typically, a filesystem's) problem entirely.
- **File storage** exposes named files organized into a hierarchy of directories, with metadata (size, permissions, timestamps) the storage layer itself tracks and updates.
- **Object storage** exposes a flat namespace of immutable, key-addressed blobs with their own metadata, deliberately **without** a hierarchy or in-place mutation.

§10 makes this comparison exhaustive; §11–§37 build all three so the difference is something you've implemented, not just described.

---

# 10. The Three Storage Abstractions, Compared Head to Head

| | Block Storage | File Storage | Object Storage |
|---|---|---|---|
| Addressed by | A block number (an integer offset) | A hierarchical path (`/home/user/doc.txt`) | A flat key (`user-42/doc.txt` — no real hierarchy, just a string that *looks* like one) |
| Mutability | In-place — overwrite any block at any time | In-place — open, seek, write, at any byte offset | **Immutable** — a "PUT" replaces the whole object, or creates a new version (§33-§34) |
| Metadata | None — the caller manages any structure | Rich, built-in (permissions, timestamps, size) | Present, but simpler and often user-defined (arbitrary key-value tags) |
| Typical consumer | A filesystem, or a database's own storage engine | An application using standard file I/O (`open`, `read`, `write`) | An application via an HTTP-based API (`PUT`/`GET`/`DELETE`) |
| Scales to | One machine (or a SAN presenting it over a network) | Practically, one machine or a tightly-coupled cluster (hierarchy makes sharding hard, §49) | Effortlessly to thousands of machines (§49-§53) — no hierarchy to keep consistent across nodes |
| Real-world example | AWS EBS, a SAN LUN, a raw `/dev/sdb` | Your laptop's filesystem, NFS, EFS | AWS S3, Google Cloud Storage, Azure Blob Storage |

---

# 11. What Block Storage Actually Is (Fixed-Size Blocks, No Metadata)

At its core, a block device is astonishingly simple: a fixed total size, divided into equal-sized **blocks** (commonly 512 bytes or 4 KB), each identified by an integer index. It supports exactly two operations — **read block N** and **write block N** — and it has **zero concept** of files, names, or even which blocks are "in use." That emptiness is deliberate: it's what lets block storage be a universal foundation any higher-level structure (a filesystem, a database's own page-based storage engine, a RAID array) can be built on top of, without the block layer imposing its own opinions about structure.

---

# 12. Real-World Examples: EBS, SAN, a Raw Disk Device

AWS EBS ("Elastic Block Store") is exactly this abstraction, network-attached: an application (or, more precisely, the guest OS's filesystem) sees what looks like a local disk device and formats it with a filesystem of its choice — EBS itself has no idea whether that filesystem is ext4, NTFS, or nothing at all. A SAN (Storage Area Network) LUN presented to a server behaves identically. This is the pattern §13 reproduces in miniature: a "disk" that only understands block numbers.

---

# 13. Phase 1 — Implementing a Minimal Block Device Abstraction in Java

```java
// block/BlockDevice.java
public interface BlockDevice {
    int getBlockSize();
    long getBlockCount();
    void readBlock(long blockNumber, byte[] destination) throws IOException;
    void writeBlock(long blockNumber, byte[] source) throws IOException;
}
```

```java
// block/FileBackedBlockDevice.java
public class FileBackedBlockDevice implements BlockDevice {
    private final RandomAccessFile file;
    private final int blockSize;
    private final long blockCount;

    public FileBackedBlockDevice(Path backingFile, int blockSize, long blockCount) throws IOException {
        this.blockSize = blockSize;
        this.blockCount = blockCount;
        this.file = new RandomAccessFile(backingFile.toFile(), "rw");
        file.setLength((long) blockSize * blockCount); // pre-allocate the full "disk" size up front
    }

    @Override
    public void readBlock(long blockNumber, byte[] destination) throws IOException {
        validateBlockNumber(blockNumber);
        file.seek(blockNumber * blockSize);
        file.readFully(destination, 0, blockSize);
    }

    @Override
    public void writeBlock(long blockNumber, byte[] source) throws IOException {
        validateBlockNumber(blockNumber);
        file.seek(blockNumber * blockSize);
        file.write(source, 0, blockSize);
    }

    private void validateBlockNumber(long blockNumber) {
        if (blockNumber < 0 || blockNumber >= blockCount) throw new IllegalArgumentException("Block out of range: " + blockNumber);
    }
    // getBlockSize()/getBlockCount() omitted for brevity
}
```

A single local file, pre-allocated to a fixed size, is a completely faithful stand-in for a real block device for every purpose this guide needs — `file.seek(blockNumber * blockSize)` is exactly the arithmetic a real disk controller performs to translate a logical block address into a physical location.

---

# 14. Reading and Writing Fixed-Size Blocks by Address

```java
BlockDevice disk = new FileBackedBlockDevice(Path.of("disk0.img"), 4096, 1000); // a 1000-block, 4KB-block "disk"
byte[] block = new byte[4096];
Arrays.fill(block, (byte) 'A');
disk.writeBlock(42, block);       // writes directly to byte offset 42 * 4096 in the backing file

byte[] readBack = new byte[4096];
disk.readBlock(42, readBack);     // reads back exactly what was written — no interpretation, no metadata, nothing else
```

Notice what's conspicuously **absent**: no filename, no notion of "this block belongs to that file," no record anywhere of which blocks are in use versus free. Every one of those concerns belongs to the layer above — exactly what §17 onward builds.

---

# 15. Follow-up Question 2 — "Why Would an Application Choose Block Storage Over a Filesystem?"

> **Interviewer:** *"If block storage is this bare, why would anything use it directly instead of just using files?"*

Because some applications — most notably **databases** — want to manage their own on-disk layout precisely, for reasons a general-purpose filesystem can't anticipate: a database knows its own page size, its own access patterns (sequential scan vs. random point lookup), and its own consistency requirements (its own write-ahead log, exactly as [the TinyDB guide's §9](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) builds), and often gets **better** performance and control by bypassing a general-purpose filesystem's abstractions and managing raw blocks directly. This is precisely why cloud databases frequently provision **raw EBS volumes**, not a mounted network filesystem, for their data directory.

---

# 16. Where Block Storage's Responsibility Ends and the Filesystem's Begins

The boundary is exact and worth stating precisely: block storage guarantees "block N contains whatever was last written to block N" — durability and addressability of raw bytes, nothing more. Everything about **meaning** — which blocks form a file, what that file is named, which blocks are free — is metadata the layer above must define, store, and keep consistent, **using the exact same block-read/block-write primitive** as any other data. §18's inode table isn't stored somewhere magically separate from the disk — it's stored in blocks, read and written through the identical `BlockDevice` interface as file content itself.

---

# 17. What File Storage Adds on Top of Blocks: Names, Hierarchy, Metadata

File storage's entire job is answering three questions block storage has no way to answer: **which blocks belong to this file** (§18–§20), **what is this file called, and where does it live in a hierarchy** (§21–§22), and **which blocks are currently free to allocate** (§23). All three are metadata problems, solved by data structures stored **on the same block device**, coexisting with the actual file content blocks.

---

# 18. The Inode Concept: Separating a File's Identity From Its Name

The foundational design decision nearly every real filesystem (ext4, XFS, NTFS's MFT records) makes: a file's **identity and metadata** (its size, its block list, its permissions) live in a fixed-size record called an **inode**, addressed by a plain integer (the inode number) — completely separate from any **name**. A directory (§21) is then just a mapping from names to inode numbers. This separation is what makes **hard links** possible (two different names pointing at the same inode, the same file) and is the conceptual foundation §22's path resolution walks through.

---

# 19. Phase 2 — Implementing a Minimal Inode Table

```java
// filesystem/Inode.java
public class Inode {
    static final int DIRECT_BLOCK_COUNT = 12; // matches classic Unix filesystem design, kept small for this guide's clarity

    int inodeNumber;
    long fileSize;
    boolean isDirectory;
    long[] directBlocks = new long[DIRECT_BLOCK_COUNT]; // block numbers holding this file's first 12 blocks of data
    long indirectBlock = -1; // §20 — a block that itself holds MORE block numbers, for files larger than 12 blocks

    byte[] serialize() { /* pack every field above into a fixed-size byte[] for on-disk storage */ return new byte[128]; }
    static Inode deserialize(byte[] data) { /* unpack back into an Inode object */ return new Inode(); }
}
```

```java
// A fixed region of the disk, reserved at filesystem-creation time, holding every inode by inode number
public class InodeTable {
    private final BlockDevice device;
    private final long firstInodeTableBlock;
    private final int inodesPerBlock;

    public Inode read(int inodeNumber) throws IOException {
        long blockNumber = firstInodeTableBlock + inodeNumber / inodesPerBlock;
        byte[] block = new byte[device.getBlockSize()];
        device.readBlock(blockNumber, block);
        int offsetWithinBlock = (inodeNumber % inodesPerBlock) * INODE_SIZE_BYTES;
        return Inode.deserialize(Arrays.copyOfRange(block, offsetWithinBlock, offsetWithinBlock + INODE_SIZE_BYTES));
    }
    // write(int inodeNumber, Inode inode) — symmetric: read the block, splice in the updated inode bytes, write it back
}
```

---

# 20. Phase 3 — Mapping a File's Logical Bytes to Physical Blocks (Direct + Indirect Pointers)

A file's first 12 blocks (§19's `directBlocks`) are found with zero extra disk reads — the block numbers sit right in the inode itself, already in memory once the inode is loaded. A **13th** block requires one more layer:

```java
long resolveBlockNumber(Inode inode, int logicalBlockIndex, BlockDevice device) throws IOException {
    if (logicalBlockIndex < Inode.DIRECT_BLOCK_COUNT) {
        return inode.directBlocks[logicalBlockIndex]; // no extra disk read — already in the inode
    }
    // Beyond the direct blocks: the indirect block holds MORE block numbers, one extra disk read to fetch them
    byte[] indirectBlockData = new byte[device.getBlockSize()];
    device.readBlock(inode.indirectBlock, indirectBlockData);
    int indexWithinIndirectBlock = logicalBlockIndex - Inode.DIRECT_BLOCK_COUNT;
    return readLongAt(indirectBlockData, indexWithinIndirectBlock * 8); // each entry is an 8-byte block number
}
```

This direct/indirect split is a genuine, deliberate space-vs-lookup-cost tradeoff: keeping every file's entire block list inline in the inode would make the inode's fixed size balloon to accommodate the largest possible file; the indirect pointer lets small files (the overwhelming majority in most real filesystems) pay **zero** extra lookup cost, while only large files pay one extra disk read per indirect block's worth of addresses (real filesystems add double- and triple-indirect blocks for even larger files, the same idea nested one or two levels deeper).

---

# 21. Phase 4 — Implementing a Directory as a Special File

```java
// filesystem/Directory.java — a directory's "content" is just a list of (name -> inode number) entries
public class DirectoryEntry {
    String name;
    int inodeNumber;
}

public class Directory {
    private final List<DirectoryEntry> entries = new ArrayList<>();

    byte[] serialize() { /* pack entries into bytes — this IS the directory inode's file content, per §17-18 */ return new byte[0]; }
    static Directory deserialize(byte[] data) { /* unpack back into entries */ return new Directory(); }

    Integer lookup(String name) {
        return entries.stream().filter(e -> e.name.equals(name)).map(e -> e.inodeNumber).findFirst().orElse(null);
    }
    void addEntry(String name, int inodeNumber) { entries.add(new DirectoryEntry(name, inodeNumber)); }
}
```

The key realization §17 sets up and this section delivers on: **a directory is not a special kind of on-disk object** — it's an ordinary inode (`isDirectory = true`) whose "file content," read through the exact same block-mapping machinery as any other file (§20), happens to be a serialized list of name-to-inode-number mappings rather than arbitrary user data.

---

# 22. Phase 5 — Path Resolution: Walking Directories to Find an Inode

```java
public Inode resolvePath(String path) throws IOException {
    Inode current = inodeTable.read(ROOT_INODE_NUMBER); // every absolute path starts at a well-known root inode
    for (String segment : path.split("/")) {
        if (segment.isEmpty()) continue; // handles a leading "/" cleanly
        if (!current.isDirectory) throw new NotADirectoryException(segment);
        Directory directory = Directory.deserialize(readEntireFile(current)); // §20's block mapping, applied to a directory's own content
        Integer nextInodeNumber = directory.lookup(segment);
        if (nextInodeNumber == null) throw new FileNotFoundException(path);
        current = inodeTable.read(nextInodeNumber);
    }
    return current;
}
```

Resolving `/home/alice/notes.txt` means: read the root inode, read its directory content, look up `"home"` to get an inode number, read *that* inode, read *its* directory content, look up `"alice"`, and so on — one directory-content read and lookup per path segment, terminating at the file's own inode. This is the mechanical answer to "how does a filesystem turn a path string into actual data" — no shortcuts, no magic, just repeated application of §20's block-mapping and §21's lookup.

---

# 23. Free Space Management: The Block Bitmap

```java
// A single bit per block: 0 = free, 1 = in use — one of the most compact possible free-space representations
public class BlockBitmap {
    private final byte[] bitmap;

    public boolean isFree(long blockNumber) {
        return (bitmap[(int) (blockNumber / 8)] & (1 << (blockNumber % 8))) == 0;
    }

    public long allocateFreeBlock() {
        for (long i = 0; i < bitmap.length * 8L; i++) {
            if (isFree(i)) { markUsed(i); return i; }
        }
        throw new IllegalStateException("Disk full");
    }

    private void markUsed(long blockNumber) {
        bitmap[(int) (blockNumber / 8)] |= (1 << (blockNumber % 8));
    }
    // markFree(long blockNumber) — the mirror-image operation, used when a file is deleted or truncated
}
```

A bitmap this simple (one bit per block, stored in its own reserved region of the disk, exactly like the inode table) is what every allocation and deallocation in this filesystem consults — `allocateFreeBlock()` is called every time a file grows past its currently-allocated blocks, and the corresponding bit must be flipped back to free the instant a block is released, or the filesystem will eventually believe it's full when it isn't (a **space leak**).

---

# 24. Follow-up Question 3 — "How Does the Filesystem Know Which Blocks Are Free?"

> **Interviewer:** *"You've mapped a file to its blocks — but how does the filesystem decide which block to hand out next when a file grows?"*

Exactly §23's bitmap, consulted before every allocation. Worth adding, if pressed further: a **linear scan** for the first free bit (as `allocateFreeBlock()` does above) is simple and correct, but real filesystems typically maintain **hints** (a cached "last block allocated near here" pointer, or a free-block count per allocation group) specifically to avoid an O(n) scan over the entire bitmap on every single allocation as a disk fills up — a concrete, defensible answer to a natural follow-up about this design's own performance ceiling.

---

# 25. Follow-up Question 4 — "What Happens If the Power Fails Mid-Write?" (Journaling)

> **Interviewer:** *"Suppose the power fails exactly between updating the inode's block list and updating the free-space bitmap to mark that new block as used. What state is the filesystem in now?"*

This is the exact filesystem-level analogue of [the TinyDB guide's crash-recovery problem](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>): a **partially applied** metadata update — a block that's referenced by an inode but *not* marked used in the bitmap (or vice versa) — leaves the filesystem's own bookkeeping internally inconsistent, potentially allowing that block to be handed out to a second file later, silently corrupting the first one. The fix is the same one that guide already derived: a **write-ahead journal**, applied here to metadata operations instead of database rows.

---

# 26. Phase 6 — A Minimal Write-Ahead Journal for Metadata Consistency

```java
public class Journal {
    private final BlockDevice device;
    private final long journalStartBlock;

    /** Records the INTENT of a multi-step metadata change before touching the real inode table or bitmap. */
    public void logTransaction(List<BlockWrite> plannedWrites) throws IOException {
        byte[] journalEntry = serializeTransaction(plannedWrites);
        device.writeBlock(nextJournalBlock(), journalEntry); // durable BEFORE any real metadata block is touched
    }

    public void applyLoggedWrites(List<BlockWrite> plannedWrites) throws IOException {
        for (BlockWrite write : plannedWrites) device.writeBlock(write.blockNumber(), write.data()); // now safe to apply for real
        markTransactionComplete();
    }

    /** Run at mount time: replay any transaction that was logged but never marked complete before the crash. */
    public void recover() throws IOException {
        for (List<BlockWrite> incompleteTransaction : readIncompleteTransactions()) {
            applyLoggedWrites(incompleteTransaction); // re-apply — idempotent, since it just rewrites the same target blocks
        }
    }
    // record BlockWrite(long blockNumber, byte[] data) {}
}
```

Both the inode update **and** the bitmap update for a single file-growth operation are logged together, as one transaction, **before** either real metadata block is touched — exactly [the WAL discipline the TinyDB guide's §9 establishes](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>), just applied to filesystem metadata blocks instead of database rows. A crash between logging and applying is recoverable (§26's `recover()` simply finishes what was already durably intended); a crash before logging never started the operation at all — there is no window where the filesystem's own bookkeeping can end up in a state that was never fully intended.

---

# 27. What Object Storage Removes: No Hierarchy, No In-Place Mutation

Object storage's design is best understood as a set of deliberate **removals** from the filesystem model just built: no directory hierarchy (a "folder" in S3's console is a cosmetic illusion drawn from key prefixes, not a real structure the storage layer maintains), no partial in-place writes (you cannot seek to byte 100 of an S3 object and overwrite 10 bytes — you replace the whole object, or append a new version), and no strong ordering/locking guarantees across objects. Every one of these removals is what §28 argues makes horizontal scale tractable in a way §17–§26's filesystem design fundamentally is not.

---

# 28. Why Object Storage Scales Differently Than File Storage

A hierarchical filesystem's directories are a **shared, mutually-dependent structure** — renaming a top-level directory conceptually touches every path beneath it, and two machines both trying to modify the same directory's contents need real coordination (locking, or something like §26's journal, generalized across machines) to avoid corrupting it. A flat key space has no such shared structure to coordinate over: `PUT("orders/2024/invoice-88.pdf", bytes)` and `PUT("users/42/avatar.png", bytes)` share **nothing** to coordinate about — they can be handled by two completely independent machines with zero communication between them. Removing hierarchy isn't a limitation imposed by S3-like systems arbitrarily; it's the specific design choice that makes near-linear horizontal scaling possible at all (§49–§53 build on this directly).

---

# 29. Phase 7 — Designing the Object Storage API (PUT/GET/DELETE/LIST by Key)

```java
public interface MiniObjectStore {
    String put(String key, byte[] data, Map<String, String> metadata) throws IOException; // returns a version id, §34
    ObjectData get(String key) throws IOException;
    ObjectData get(String key, String versionId) throws IOException;                       // §34
    void delete(String key) throws IOException;
    List<String> listKeys(String keyPrefix);                                               // the "folder" illusion, §27
}
```

Four operations, no `mkdir`, no `rename`, no `seek` — this narrow surface is the entire object-storage contract, and it's narrow specifically **because** removing everything else is what §28 argued for.

---

# 30. Phase 8 — Content-Addressable Storage: Keying Objects by Hash

A common, robust implementation technique: store each object's actual bytes on disk under a filename derived from a **cryptographic hash of its content**, completely independent of the caller-supplied key:

```java
public class ContentAddressableStorage {
    public String store(byte[] data) throws IOException, NoSuchAlgorithmException {
        MessageDigest sha256 = MessageDigest.getInstance("SHA-256");
        byte[] hash = sha256.digest(data);
        String contentId = HexFormat.of().formatHex(hash); // a fixed-length, collision-resistant filename
        Path objectPath = pathFor(contentId);
        if (!Files.exists(objectPath)) { // identical content is stored ONCE, regardless of how many keys point to it
            Files.write(objectPath, data);
        }
        return contentId;
    }
    private Path pathFor(String contentId) {
        return Path.of("objects", contentId.substring(0, 2), contentId.substring(2)); // fan out into subdirectories — avoids millions of files in one directory
    }
}
```

Deriving the storage location from a hash of the content itself — rather than from the caller's key — has a genuinely useful side effect: **deduplication is automatic**. Two different keys that happen to point at byte-for-byte identical content are stored on disk exactly once, since they hash to the same content ID, at zero extra implementation cost.

---

# 31. Phase 9 — Implementing PUT: Writing Immutable Objects

```java
public class MiniObjectStoreImpl implements MiniObjectStore {
    private final ContentAddressableStorage contentStorage;
    private final ObjectIndex index; // §37 — the separate catalog mapping key -> content id + metadata

    @Override
    public String put(String key, byte[] data, Map<String, String> metadata) throws IOException {
        String contentId = contentStorage.store(data);       // §30 — write the bytes once, immutably
        String versionId = UUID.randomUUID().toString();     // §34 — this PUT becomes a NEW version, not an overwrite
        index.recordVersion(key, versionId, contentId, metadata, Instant.now());
        return versionId;
    }
}
```

Nothing about an existing object's stored bytes is ever touched by a subsequent `put()` to the same key — a new version is simply **added** to the index (§37), pointing at its own, separately content-addressed bytes. This is the concrete mechanism behind §27's "no in-place mutation" claim: mutation, from the object store's own point of view, doesn't exist — only new immutable writes and index updates pointing at them do.

---

# 32. Phase 10 — Implementing GET and LIST

```java
@Override
public ObjectData get(String key) throws IOException {
    ObjectVersion latest = index.getLatestVersion(key); // §37
    if (latest == null) throw new NoSuchKeyException(key);
    byte[] data = contentStorage.retrieve(latest.contentId());
    return new ObjectData(data, latest.metadata());
}

@Override
public List<String> listKeys(String keyPrefix) {
    return index.allKeys().stream()
        .filter(key -> key.startsWith(keyPrefix)) // the ENTIRE mechanism behind S3's "folder" illusion, §27
        .sorted()
        .toList();
}
```

`listKeys("orders/2024/")` returning every key that happens to start with that string is, precisely, how S3's console draws something that looks like a folder browser — there is no real directory object being traversed, only a string-prefix filter over a flat key space, which is the mechanical proof of §27's claim that hierarchy here is cosmetic.

---

# 33. Follow-up Question 5 — "Objects Are Immutable — How Do You 'Update' One?"

> **Interviewer:** *"If an object can't be modified in place, how does 'updating a file in S3' actually work when someone re-uploads it?"*

It doesn't update anything — it **replaces** the key's pointer to point at a brand-new, independently-stored version (§31), and (if versioning is enabled, §34) the **old** version's bytes remain fully retrievable by their own version ID, completely unaffected by the new `PUT`. This is a genuinely different mental model from a filesystem's `open`-`seek`-`write`, and it's the correct answer to give explicitly rather than glossing over — "you PUT a new version" is a categorically different operation from "you edited the file."

---

# 34. Phase 11 — Object Versioning

```java
// objectstore/ObjectVersion.java
public record ObjectVersion(String key, String versionId, String contentId, Map<String, String> metadata, Instant createdAt) { }

// objectstore/ObjectIndex.java — the append-only version history behind every key
public class ObjectIndex {
    private final Map<String, List<ObjectVersion>> versionsByKey = new ConcurrentHashMap<>();

    public void recordVersion(String key, String versionId, String contentId, Map<String, String> metadata, Instant now) {
        versionsByKey.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>())
            .add(new ObjectVersion(key, versionId, contentId, metadata, now));
    }

    public ObjectVersion getLatestVersion(String key) {
        List<ObjectVersion> versions = versionsByKey.get(key);
        return (versions == null || versions.isEmpty()) ? null : versions.get(versions.size() - 1);
    }

    public ObjectVersion getVersion(String key, String versionId) {
        return versionsByKey.getOrDefault(key, List.of()).stream()
            .filter(v -> v.versionId().equals(versionId)).findFirst().orElse(null);
    }
}
```

`versionsByKey` is genuinely **append-only** — every `put()` (§31) adds a new `ObjectVersion`, nothing is ever removed from this list by an ordinary write, which is exactly what makes "restore a previous version" a read against already-durable history rather than a special recovery procedure.

---

# 35. Follow-up Question 6 — "How Would You Store an Object Larger Than a Single Disk?" (Multipart Upload, Chunking)

> **Interviewer:** *"A user wants to upload a 500GB video file. Your storage node has a 200GB disk. What now?"*

Split the object into fixed-size **parts** at upload time, store each part independently (potentially on **different** disks/machines entirely), and record the ordered list of part identifiers as the object's real content descriptor — the object's "bytes," from the index's perspective, becomes a sequence of part references rather than one contiguous blob that must physically fit in one place.

---

# 36. Phase 12 — Multipart Upload and Chunked Storage

```java
public class MultipartUpload {
    private final String uploadId = UUID.randomUUID().toString();
    private final List<String> partContentIds = new ArrayList<>(); // in order — part 0, part 1, part 2, ...
    private final ContentAddressableStorage contentStorage;

    public void uploadPart(int partNumber, byte[] partData) throws IOException {
        String contentId = contentStorage.store(partData); // §30 — each part is its own content-addressed object
        while (partContentIds.size() <= partNumber) partContentIds.add(null);
        partContentIds.set(partNumber, contentId);
    }

    public String complete(String key, ObjectIndex index) {
        String manifestContentId = storeManifest(partContentIds); // a small object listing every part's content id, IN ORDER
        String versionId = UUID.randomUUID().toString();
        index.recordVersion(key, versionId, manifestContentId, Map.of("multipart", "true"), Instant.now());
        return versionId;
    }
    // storeManifest(...) omitted for brevity — serializes partContentIds and stores it via contentStorage, same as any object
}
```

A `get()` on a multipart object (§32, extended) first fetches the small **manifest** object, then fetches and concatenates each listed part in order — the caller sees one contiguous byte stream, with the actual physical storage split arbitrarily across parts (and, in a real distributed system, across machines) underneath. This is exactly how S3's real multipart upload API works, including the detail that individual parts can be uploaded **in parallel, out of order, and even retried independently** on failure, since each part is a wholly independent content-addressed write.

---

# 37. Metadata and a Separate Index: Why Object Storage Needs Its Own Catalog

Notice that `ObjectIndex` (§34) — the structure mapping keys to their version history and content IDs — is functionally identical in spirit to §18's inode table: it's the **metadata layer** that gives meaning to otherwise-anonymous, content-addressed blobs (§30). The crucial architectural difference from a filesystem's inode table is that this index has **no** hierarchy to maintain consistent (§28) — it's a flat map, `key -> [versions]`, which is precisely what makes it tractable to shard and replicate across many machines (§49–§51) in a way an inode table's directory tree fundamentally resists.

---

# 38. Follow-up Question 7 — "A Single Disk Will Eventually Fail — How Do You Protect Against That?"

> **Interviewer:** *"Everything we've built so far — the filesystem, the object store — assumes the underlying block device just works. Real disks fail. How do you protect against that, and what's the actual tradeoff between the different RAID levels people always name-drop?"*

This is the pivot from "storage abstractions" to "storage reliability" — and it's answered concretely, not by naming acronyms: RAID's entire idea is spreading data across **multiple physical disks** in a pattern that lets the array survive losing one (or, for some levels, more than one) disk **without losing any data**, at some cost in usable capacity, write performance, or both. §39–§47 build each level's actual mechanism, specifically so "RAID 5 uses parity" becomes something you've implemented, not memorized.

---

# 39. RAID 0: Striping for Performance, No Redundancy

**RAID 0** splits data into stripes, written round-robin across N disks — block 0 on disk A, block 1 on disk B, block 2 on disk A, and so on. This means a large read/write can happen **in parallel** across all N disks simultaneously, multiplying throughput by roughly N. The critical, honest caveat: RAID 0 provides **zero** redundancy — losing *any one* disk loses the *entire* array's data, since every file's blocks are spread across all disks with no duplication or recovery information whatsoever. It's pure performance, at the cost of *worse* reliability than a single disk (N disks means N times the chance that *some* disk in the array fails).

---

# 40. RAID 1: Mirroring for Redundancy, No Performance Gain on Writes

**RAID 1** takes the opposite approach: every block is written **identically** to two (or more) disks. Losing one disk loses nothing — the mirror has a complete, independent copy. The cost is capacity (two disks' worth of hardware provides only one disk's worth of usable space) and, for writes, no speedup at all (every write must complete on every mirror), though **reads** can actually be served from whichever mirror is least busy, a real throughput win RAID 1 does offer.

---

# 41. Phase 13 — Implementing RAID 0 Striping From Scratch

```java
// raid/Raid0StripedDevice.java
public class Raid0StripedDevice implements BlockDevice {
    private final BlockDevice[] disks;

    @Override
    public void writeBlock(long logicalBlockNumber, byte[] data) throws IOException {
        int diskIndex = (int) (logicalBlockNumber % disks.length);         // round-robin across disks
        long physicalBlockNumber = logicalBlockNumber / disks.length;       // this disk's own local block address
        disks[diskIndex].writeBlock(physicalBlockNumber, data);
    }

    @Override
    public void readBlock(long logicalBlockNumber, byte[] destination) throws IOException {
        int diskIndex = (int) (logicalBlockNumber % disks.length);
        long physicalBlockNumber = logicalBlockNumber / disks.length;
        disks[diskIndex].readBlock(physicalBlockNumber, destination);
    }

    @Override
    public long getBlockCount() { return Arrays.stream(disks).mapToLong(BlockDevice::getBlockCount).sum(); }
}
```

`logicalBlockNumber % disks.length` (which disk) and `logicalBlockNumber / disks.length` (which local block on that disk) is the entire striping algorithm — every caller of this class sees one large, fast `BlockDevice` (§13's same interface), with the striping completely transparent underneath, exactly the way §16 argued a layer's responsibility should be hidden from the layer above it.

---

# 42. Phase 14 — Implementing RAID 1 Mirroring From Scratch

```java
// raid/Raid1MirroredDevice.java
public class Raid1MirroredDevice implements BlockDevice {
    private final BlockDevice[] mirrors;
    private int nextReadMirror = 0; // simple round-robin read load balancing across mirrors

    @Override
    public void writeBlock(long blockNumber, byte[] data) throws IOException {
        for (BlockDevice mirror : mirrors) {
            mirror.writeBlock(blockNumber, data); // EVERY mirror gets the identical write — no speedup, full redundancy
        }
    }

    @Override
    public void readBlock(long blockNumber, byte[] destination) throws IOException {
        int mirrorIndex = nextReadMirror;
        nextReadMirror = (nextReadMirror + 1) % mirrors.length; // spread read load — the one real perf win RAID 1 offers
        try {
            mirrors[mirrorIndex].readBlock(blockNumber, destination);
        } catch (IOException failedMirror) {
            readFromAnyOtherWorkingMirror(blockNumber, destination, mirrorIndex); // degraded-mode read — the array survives this
        }
    }
    // readFromAnyOtherWorkingMirror(...) — tries every remaining mirror until one succeeds
}
```

The `catch` block in `readBlock` is the entire point of RAID 1 made concrete: a failed mirror doesn't fail the read at all, it just means one fewer disk to round-robin across — exactly the "the array survives one disk's failure" guarantee stated in prose becomes a real, testable code path here.

---

# 43. RAID 5: Striping With Distributed Parity

RAID 5 combines striping's performance (§39) with real redundancy, at a much better capacity cost than mirroring (§40): across N disks, N−1 disks' worth of blocks hold real data, and the **Nth** — a role that **rotates** across every disk, rather than being fixed to one — holds a **parity** block computed from the others. Losing any **single** disk (data or parity) is fully recoverable: the missing disk's contents are reconstructed by XOR-ing the survivors together (§44).

---

# 44. Deriving XOR Parity: How One Extra Block Recovers Any One Failed Disk

The bitwise XOR (`^`) operator has a property that makes it perfect for this: `A ^ B ^ C ^ P = 0` if `P` is defined as `P = A ^ B ^ C` (XOR-ing anything with itself cancels to zero, and `X ^ 0 = X`). Rearranging that equation for any single missing term recovers it from the rest:

$$P = A \oplus B \oplus C \qquad\Longrightarrow\qquad A = B \oplus C \oplus P$$

This holds **regardless of which one of the four values (A, B, C, or P itself) goes missing** — XOR-ing together whichever three survive always reconstructs the fourth exactly. This single algebraic fact, applied byte-by-byte across every disk in the stripe, is the *entire* mathematical mechanism behind RAID 5's fault tolerance — there's no more advanced math hiding underneath it.

```java
// raid/XorParity.java
public class XorParity {
    public static byte[] compute(byte[]... blocks) {
        byte[] parity = new byte[blocks[0].length];
        for (byte[] block : blocks) {
            for (int i = 0; i < block.length; i++) parity[i] ^= block[i];
        }
        return parity;
    }
    // reconstruct(byte[]... survivingBlocksIncludingParity) is IDENTICAL code — XOR treats every input the same way,
    // which is exactly why "compute parity" and "reconstruct a missing block" are literally the same function.
}
```

That last comment is worth internalizing on its own: **computing parity and reconstructing a lost block from parity are the same operation**, applied to a different subset of "the disks I currently have data for" — there is no separate "recovery algorithm" to implement beyond calling this identical function with a different set of surviving inputs.

---

# 45. Phase 15 — Implementing RAID 5 Parity Calculation and Reconstruction

```java
// raid/Raid5Device.java
public class Raid5Device implements BlockDevice {
    private final BlockDevice[] disks; // N disks; one per stripe row holds parity, rotating which one

    @Override
    public void writeBlock(long stripeNumber, byte[][] dataBlocksForThisStripe) throws IOException {
        int parityDiskIndex = (int) (stripeNumber % disks.length); // rotates — spreads the parity-write load evenly, unlike a fixed parity disk
        byte[] parity = XorParity.compute(dataBlocksForThisStripe);
        int dataIndex = 0;
        for (int disk = 0; disk < disks.length; disk++) {
            byte[] blockToWrite = (disk == parityDiskIndex) ? parity : dataBlocksForThisStripe[dataIndex++];
            disks[disk].writeBlock(stripeNumber / disks.length, blockToWrite);
        }
    }

    public byte[] readWithReconstruction(long stripeNumber, int failedDiskIndex) throws IOException {
        List<byte[]> survivingBlocks = new ArrayList<>();
        for (int disk = 0; disk < disks.length; disk++) {
            if (disk == failedDiskIndex) continue; // skip the failed disk entirely
            byte[] block = new byte[disks[disk].getBlockSize()];
            disks[disk].readBlock(stripeNumber / disks.length, block);
            survivingBlocks.add(block);
        }
        return XorParity.compute(survivingBlocks.toArray(new byte[0][])); // §44 — reconstruction IS computation
    }
}
```

Rotating which physical disk holds parity for each stripe (rather than dedicating one fixed disk to parity forever, which real RAID 4 does and RAID 5 specifically improves on) evenly spreads the extra write work every parity update requires across every disk in the array, avoiding a single disk becoming a write bottleneck under sustained load.

---

# 46. RAID 6: Surviving Two Simultaneous Disk Failures

RAID 5's XOR reconstruction (§44) works for exactly **one** missing block per stripe — the equation `A = B ⊕ C ⊕ P` has no solution if *two* of the four values are unknown simultaneously. RAID 6 adds a **second**, independently-computed parity block per stripe (typically using Reed-Solomon coding rather than a second XOR, since a second plain XOR parity wouldn't actually be independent enough to solve two simultaneous unknowns) — at the cost of two disks' worth of capacity dedicated to parity instead of one, RAID 6 tolerates **any two** simultaneous disk failures, which matters increasingly as individual disk capacities grow (a larger disk takes longer to rebuild after one failure, extending the window during which a *second* failure would be catastrophic under RAID 5 but survivable under RAID 6).

---

# 47. RAID 10: Combining Mirroring and Striping

RAID 10 (sometimes written RAID 1+0) mirrors pairs of disks (§40, §42), then stripes (§39, §41) data **across** those mirrored pairs — getting striping's read/write performance **and** mirroring's straightforward, computation-free redundancy (no XOR reconstruction needed at all; a failed disk's mirror just takes over directly, per §42's `catch` block) at the cost of the same 50% usable-capacity penalty pure mirroring has. It's a common choice specifically because its recovery path (§42) is simpler and faster than RAID 5/6's parity reconstruction (§44–§46), at a real, accepted capacity cost.

---

# 48. Follow-up Question 8 — "Which RAID Level Would You Choose for a Database vs a Video Archive?"

> **Interviewer:** *"Given everything we just built — which RAID level would you actually pick for a transactional database's data directory, versus a cold archive of raw video footage, and why?"*

- **A transactional database** typically wants **RAID 10**: databases issue many small, latency-sensitive random writes (index updates, row updates), and RAID 5/6's parity recomputation on every write (the "read-modify-write" penalty — updating one data block requires reading the old data and old parity, then writing new data and new parity) directly hurts exactly that access pattern; RAID 10's mirrored writes have no such penalty.
- **A cold, rarely-written video archive** is a strong fit for **RAID 6** (or even RAID 5, depending on the durability bar): writes are infrequent and typically sequential/bulk (so the read-modify-write penalty matters far less), reads are the common case, and the capacity efficiency of parity-based redundancy (losing only 1-2 disks' worth of capacity to redundancy, regardless of how many disks are in the array) matters far more than for a smaller, latency-critical database volume.

The right answer is never "RAID X is best" in the abstract — it's always this specific reasoning: what's the write pattern, what's the capacity math, and what failure tolerance does the data's actual value justify paying for.

---

# 49. Follow-up Question 9 — "What If One Machine Isn't Enough? How Do You Scale Object Storage Horizontally?"

> **Interviewer:** *"RAID protects against a disk failing. What protects against an entire machine failing? And how do you serve more traffic than one machine's network interface can handle?"*

This is the natural escalation once single-machine reliability (RAID) is established — the same two problems RAID solved (spread the risk, spread the load) reappear one level up, now across **machines** instead of disks, and §27–§28's flat, hierarchy-free key space is exactly what makes this tractable rather than a nightmare of distributed directory-tree coordination.

---

# 50. Consistent Hashing: Distributing Objects Across Many Nodes

The most common technique for deciding **which machine** owns a given key: hash the key (the same SHA-256-style hashing already used for content addressing, §30) onto a large numeric ring, hash each storage node's identifier onto the same ring, and assign each key to the **next node clockwise** from its position. The payoff over naive `hash(key) % nodeCount`: adding or removing one node only reshuffles the keys immediately adjacent to it on the ring — **not** every key in the system, which is precisely the property that makes horizontally scaling an object store (adding a node without re-shuffling the entire dataset) practical at all.

---

# 51. Replication Across Nodes: Beyond RAID's Single-Machine Redundancy

Once keys are distributed (§50), each key's data is typically stored on **N** nodes (commonly 3), not just one — the exact same mirroring idea as RAID 1 (§40, §42), just applied across machines/data-centers instead of disks in one chassis, and tolerating an entire **machine's** (or even an entire **data center's**) failure, not merely a disk's. A write is usually acknowledged once a **quorum** of replicas (e.g., 2 of 3) confirm it, the same quorum-based durability/latency tradeoff [the TinyDB guide's replication sections](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) already cover in depth for database rows, now applied to object bytes.

---

# 52. Erasure Coding: RAID's Idea, Generalized Across Machines (Reed-Solomon, Conceptually)

Full replication (§51) is simple but capacity-expensive — 3x replication means storing 3 full copies of everything. **Erasure coding** generalizes §44's XOR-parity idea to a much larger scale: split an object into K data fragments, compute M additional parity fragments (via Reed-Solomon coding — a real generalization of the same "solve for the missing piece" algebra XOR demonstrates for the simplest case), and distribute all K+M fragments across different nodes. Any **K** of the K+M fragments (data or parity, in any combination) suffice to reconstruct the original object — tolerating up to M simultaneous node failures while storing only `(K+M)/K` times the original data, a dramatically better capacity ratio than full replication for the same fault tolerance (e.g., 10 data + 4 parity fragments tolerates 4 simultaneous failures at only 1.4x storage overhead, versus 4x for quadruple replication).

---

# 53. Follow-up Question 10 — "How Does S3 Achieve 11 Nines of Durability?"

> **Interviewer:** *"AWS advertises 99.999999999% durability for S3. Walk me through, mechanically, how a system actually gets a number like that."*

The honest, defensible answer combines every mechanism this Part built, stacked: **erasure coding across many independent machines and data centers** (§52) means an object survives multiple simultaneous hardware failures, not just one; **replicating across physically separate facilities** means even a full data-center-level event (power, fire, flooding) doesn't threaten every copy at once; and **continuous background verification** (periodically re-reading stored fragments and checksums, comparable in spirit to [TinyDB's own checksum discipline](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>)) catches and repairs "bit rot" — silent, gradual data corruption on a drive — before it accumulates into unrecoverable loss. The number itself is a probabilistic calculation over these independent, redundant failure domains — not a guarantee that nothing ever fails, but that the *combination* of redundancy and continuous repair makes correlated, unrecoverable loss astronomically unlikely.

---

# 54. Follow-up Question 11 — "What About Google File System / HDFS? How Do They Fit In?"

> **Interviewer:** *"Everything we just covered — consistent hashing, replication, erasure coding — describes how S3-style object stores scale. Google File System and HDFS are also distributed storage systems, but they're not built that way at all. Walk me through how they actually work, and why."*

This is a genuinely fair pushback: GFS (Google's original 2003 paper) and HDFS (its open-source, Hadoop-ecosystem descendant) are a **fourth** storage paradigm this guide hasn't touched — neither a local POSIX filesystem (§17–§26), nor a flat, hash-distributed object store (§27–§37, §49–§53), but a **distributed file system with a centralized metadata master**, purpose-built for a very specific workload: enormous files, sequential reads, append-mostly writes, running batch analytics (originally MapReduce) directly alongside the data.

---

# 55. GFS/HDFS Architecture: A Master/NameNode and Many Chunkservers/DataNodes

Both systems split responsibility along exactly the same line: one (logically) centralized **master** (called the **Master** in GFS, the **NameNode** in HDFS) holds **all metadata** — the directory tree, which files exist, and which chunks/blocks make up each file — entirely in memory, for speed. The actual file **data** lives on a large fleet of **chunkservers** (GFS) or **DataNodes** (HDFS), which store raw chunks on their own local disks and know nothing about the directory hierarchy at all — a DataNode is, from its own point of view, just a dumb container for numbered chunks, philosophically close to §11's bare block device, just replicated (§57) instead of RAID-protected.

```text
                        Client
                          |
                 1. "where are the chunks
                     for /data/logs/2024.log?"
                          v
                    Master / NameNode
                 (all metadata, in memory)
                          |
                 2. "chunks C1, C2, C3 are on
                     DataNodes [A,B], [B,C], [A,C]"
                          v
                        Client
                          |
                 3. reads/writes chunk data DIRECTLY --------> DataNode A, B, C
                    (the master is NEVER in this data path)      (chunkservers)
```

---

# 56. Why Large, Fixed-Size Chunks (64MB-128MB) Instead of 4KB Blocks

§13's block device used small, fixed-size blocks (4KB) — appropriate for a general-purpose filesystem serving arbitrary small files. GFS/HDFS instead use **enormous** chunks — 64MB in the original GFS paper, 128MB by default in HDFS — for a reason directly tied to the master's design (§55): every chunk needs a metadata entry in the master's **in-memory** table, and a smaller chunk size means proportionally more chunks (and more metadata) for the same total data, directly threatening the one resource (the master's RAM) this whole architecture is built around conserving. A 64MB chunk size for a workload of genuinely enormous files (web crawls, log archives, scientific datasets — GFS/HDFS's actual target workload) keeps the *number* of chunks, and therefore the metadata burden on the master, manageable even at petabyte scale.

---

# 57. Separating Control Plane From Data Plane: Metadata via the Master, Bulk Transfer Directly Between Client and Chunkservers

§55's diagram makes the single most important architectural decision in this design visible: the master answers **"where"** (step 2), but every byte of actual file data (step 3) flows directly between the client and the chunkservers/DataNodes — **never through the master at all**. This separation of a lightweight "control plane" (metadata lookups, small and fast) from a heavyweight "data plane" (bulk byte transfer, large and slow) is exactly what lets the master remain a single logical service without becoming a throughput bottleneck: it answers metadata queries all day without ever having to move a single byte of the petabytes of data it's the source of truth for.

---

# 58. The Write Pipeline: Replicating a Chunk Across Multiple DataNodes

A write in this design is deliberately **pipelined** rather than fanned out from the client to every replica independently: the client sends the chunk's data to the **first** DataNode in the replica set, which forwards it to the **second** while simultaneously writing its own local copy, which forwards it to the **third**, and so on — a chain, not a star. This matters for the client's own upload bandwidth specifically: the client only ever sends the data **once**, to one DataNode, rather than needing enough upstream bandwidth to push a full copy to every replica (typically 3) simultaneously — the replication fan-out cost is paid between DataNodes, on the (typically much faster) internal cluster network, not on the client's own connection.

---

# 59. The NameNode as a Single Point of Failure — and How It's Mitigated

§55's "logically centralized" master is a real, honestly-named weakness: if the NameNode goes down, the **entire filesystem** becomes unreachable — every DataNode may be perfectly healthy, but with no metadata service to ask "where are my chunks," a client can't locate a single byte of actual data. Two complementary mitigations address this directly:

- **Checkpointing** (a Secondary NameNode in classic HDFS, "shadow masters" in GFS) periodically snapshots the in-memory metadata to disk, so a restarted NameNode doesn't have to replay its *entire* operation history from scratch — only whatever happened since the last checkpoint, the exact same "snapshot plus replay the remaining log" idea [TinyDB's §13](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) already uses for bounding its own crash-recovery time.
- **HDFS High Availability** goes further, running a genuine **hot standby** NameNode with synchronously-replicated metadata edits (via a shared, quorum-based edit log), so a failover can happen in seconds rather than requiring a full cold restart and checkpoint replay — the same "don't have just one of the thing everything depends on" instinct behind RAID (§38) and replication (§51), applied to the metadata service itself rather than to data.

---

# 60. Centralized Metadata vs Consistent Hashing: Two Different Answers to "Where Is My Data"

This is the direct, concrete contrast the interviewer's question in §54 is really probing for — two fundamentally different answers to the exact same underlying problem ("given a name, which machine has the data"):

| | Consistent hashing (§50, S3-style) | Centralized master (§55, GFS/HDFS-style) |
|---|---|---|
| Who knows "where" | **No single node** — any node can compute it from the key's hash | **One logical service** (with hot-standby failover, §59) holds the complete answer |
| Adding a node | Reshuffles only the keys near it on the ring (§50) — no coordination needed with a central authority | The master must be told about the new node and immediately starts placing new chunks there — a coordinated, not autonomous, decision |
| Failure mode | No single point of failure by design — the hashing function itself never goes down | A real, if mitigated, single point of failure (§59) |
| Best fit | Enormous numbers of small-to-medium, independently-accessed objects (§27–§28's flat key space) | Enormous **files** (not enormous *counts* of small objects), accessed by a comparatively modest number of large-scale batch/analytics clients that can tolerate a metadata-service round trip per file open |

Neither design is objectively superior — they're optimized for genuinely different access patterns (many independent small objects vs. fewer, much larger files accessed by a smaller number of heavy, sequential-read clients), which is exactly why S3 and HDFS coexist as *different* tools in real infrastructure rather than one having replaced the other.

---

# 61. Write-Once, Read-Many: Why HDFS Doesn't Support Random Writes

The final piece that makes sense of GFS/HDFS's design choices as a coherent whole rather than a list of unrelated decisions: the workload it was built for — batch analytics over enormous datasets — reads sequentially, writes rarely (often only by appending), and essentially never needs to modify an arbitrary byte offset in the middle of an existing file the way a general-purpose filesystem's `seek`-then-`write` does (§17). HDFS's actual API reflects this honestly: files are traditionally **write-once** (created, written sequentially, then closed and treated as immutable for reads) with append support added later, and in-place random writes were never a supported operation at all. This is the same "narrow the contract to match the actual workload" discipline §27's object storage design already demonstrated (immutable objects, no `seek`) — GFS/HDFS reaches a philosophically similar conclusion (avoid general-purpose mutable-file semantics) from a completely different starting architecture, which is worth recognizing as a pattern in its own right: **narrowing what a storage system allows you to do is often exactly what makes it possible to scale it.**

---

# 62. Full Worked Example: Storing a File Through All Three Layers

Tracing one `PUT` of a user's uploaded photo through every layer this guide built:

```text
Application: objectStore.put("users/42/avatar.png", photoBytes, metadata)
   |
   v
MiniObjectStoreImpl.put()                                                    (§31)
   -> ContentAddressableStorage.store(photoBytes) -> SHA-256 content id       (§30)
   -> ObjectIndex.recordVersion("users/42/avatar.png", newVersionId, ...)     (§34)
   -> (if > threshold size) split into parts via MultipartUpload             (§35-36)
   |
   v
The content-addressed bytes are ultimately written through a BlockDevice     (§13)
   |
   v
That BlockDevice is actually a Raid5Device                                   (§45)
   -> computes XOR parity across the stripe                                  (§44)
   -> writes data blocks + parity block across N physical disks              (§39, §43)
   |
   v
Physical disks — any ONE of which could fail without losing the photo
```

Every layer this guide built — content addressing, versioned metadata, multipart chunking, block-level striping, and parity-based redundancy — appears in this one upload, each solving a distinct problem the layer below it left unsolved.

---

# 63. Final Architecture

```text
                          Application
                              |
              +----------------+----------------+
              v                                 v
    MiniFileSystem (§17-26)             MiniObjectStore (§27-37)
    path -> inode -> blocks              key -> version -> content id
    directories, journal                 immutable, versioned, multipart
              |                                 |
              +----------------+----------------+
                              v
                    BlockDevice interface (§13)
                              |
              +----------------+----------------+
              v                v                v
        Raid0 (§41)      Raid1 (§42)      Raid5/6 (§45-46)
        striping         mirroring        striping + parity
              |                |                |
              +----------------+----------------+
                              v
                       Physical Disks

  Beyond one machine (§49-53):
  Consistent hashing routes keys to nodes -> replication/erasure coding
  across nodes provides RAID's guarantee at data-center scale

  A different answer to the same problem (§54-61):
  GFS/HDFS -- a centralized Master/NameNode holds all metadata in memory;
  clients ask it "where," then talk DIRECTLY to chunkservers/DataNodes for data
```

---

# 64. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Layered Architecture** | Block storage → filesystem/object storage → application, each layer only depending on the interface below it | Throughout — most explicitly §16, §28 |
| **Strategy** | Swapping which `BlockDevice` implementation (plain, RAID 0, RAID 1, RAID 5) backs a filesystem or object store, with zero change above | §13, §41-§45 |
| **Content-Addressable Storage** (a named pattern in its own right) | Deduplication and immutability in the object store | §30 |
| **Write-Ahead Logging** | Crash-consistent filesystem metadata | §25-§26, reused directly from [the TinyDB guide's §9](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) |
| **Consistent Hashing** | Distributing object keys across many nodes with minimal reshuffling on scale changes | §50 |
| **Control Plane / Data Plane Separation** | The Master/NameNode answers "where" without ever moving data itself; clients then transfer bytes directly with chunkservers/DataNodes | §55, §57 |

---

# 65. Common Mistakes

- **Mistake 1 — Treating "block storage," "file storage," and "object storage" as marketing labels instead of mechanical designs.** Each one makes a specific, derivable set of tradeoffs (§10) — an interview answer that can't explain *why* object storage doesn't support in-place edits (§27) is reciting, not reasoning.
- **Mistake 2 — Forgetting the read-modify-write penalty when recommending RAID 5/6 for a write-heavy, small-random-write workload (§48).** A database's own performance can suffer badly under a RAID level chosen purely for its capacity efficiency.
- **Mistake 3 — Confusing RAID (single-machine disk redundancy) with replication (multi-machine redundancy) as if they solve the same problem.** RAID 1 protects against a disk failing; it does nothing if the entire machine (or data center) hosting that RAID array goes offline — that's §51's job, at a different layer entirely.
- **Mistake 4 — Skipping the free-space bitmap update atomicity problem (§25).** A filesystem that updates an inode's block list and the free-space bitmap as two separate, unlogged writes can corrupt its own bookkeeping on a crash, exactly the bug §26's journal exists to prevent.
- **Mistake 5 — Assuming erasure coding and replication are interchangeable in every situation (§51-§52).** Erasure coding's capacity efficiency comes at a real CPU/reconstruction-latency cost full replication doesn't have — a system tuned for read latency over cold-storage cost efficiency might correctly choose replication despite its worse storage ratio.

---

# 66. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| Block device (§13) | Reading back exactly what was written, at every valid block number; out-of-range access throws | Direct unit tests against `FileBackedBlockDevice` |
| Filesystem path resolution (§22) | A deeply nested path resolves to the correct inode; a missing path segment throws `FileNotFoundException` | Build a small directory tree, assert resolution at each level |
| Journal crash recovery (§26) | A transaction logged but never marked complete is correctly replayed on `recover()` | Simulate a crash mid-transaction (stop before `applyLoggedWrites` completes), restart, assert the metadata ends up consistent |
| Object versioning (§34) | Every `put()` to the same key preserves the previous version, retrievable by its own version id | `put` the same key three times, assert `getVersion` returns each distinct version's own content |
| RAID 5 reconstruction (§45) | Data read via `readWithReconstruction` after simulating a disk failure exactly matches the original data | Write known data across the array, "fail" one disk (skip it), reconstruct, compare byte-for-byte against the original |

The RAID 5 reconstruction test is the one most worth taking seriously — it's the test that actually proves the XOR math (§44) was implemented correctly, not just that the array "seems to work" under the happy path where no disk ever fails.

---

# 67. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Double- and triple-indirect blocks | Filesystem support for files far larger than a single indirect block's addressing range | Extends §20's block-resolution logic one or two levels deeper |
| Hard and symbolic links | Multiple names for one inode, or a name that points at another path | Built directly on §18's identity/name separation |
| Real Reed-Solomon erasure coding | Actual RAID 6 / distributed erasure coding, not just the conceptual description in §46/§52 | Replaces XOR (§44) with proper Galois-field arithmetic for tolerating more than one simultaneous failure |
| Object lifecycle policies | Automatically transitioning old object versions to cheaper, slower storage, or expiring them | A background job consulting §34's version history's `createdAt` timestamps |
| Read-repair for erasure-coded data | Detecting and fixing a corrupted fragment during an ordinary read, not just during background scans | Extends §53's continuous-verification idea into the hot read path itself |
| A real network protocol front-end | Exposing the object store over actual HTTP, matching S3's real API surface | A thin web layer (reusable ideas from [the MiniTomcat guide](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>)) in front of §29's `MiniObjectStore` interface |

---

# 68. Progressive Interview Question Set

**Level 1 — The three abstractions**
1. Explain the mechanical (not just nominal) difference between block, file, and object storage.
2. Why can't object storage support an in-place `seek`-and-overwrite the way a filesystem does?

**Level 2 — Filesystem internals**
3. Walk through resolving `/a/b/c.txt` to its inode, step by step.
4. Why does an inode separate a file's block list from its name, and what real feature (hard links) does that separation enable?
5. What specifically can go wrong if a crash happens between an inode update and a free-space-bitmap update, and how does journaling prevent it?

**Level 3 — Object storage internals**
6. What does content-addressable storage buy you for free, as a side effect, that you didn't have to implement separately?
7. Walk through what happens, end to end, when a 10GB file is uploaded via multipart upload.

**Level 4 — RAID**
8. Derive the XOR parity equation and explain why the same function computes parity AND reconstructs a missing block.
9. Why does RAID 5 fail to tolerate two simultaneous disk failures, and what does RAID 6 change to fix that?
10. For a given workload description, justify a specific RAID level choice using the read/write pattern and capacity tradeoffs, not a memorized rule.

**Level 5 — Scaling beyond one machine**
11. Explain consistent hashing's advantage over `hash(key) % nodeCount` when a node is added or removed.
12. Compare replication and erasure coding on capacity efficiency versus reconstruction cost, for the same fault-tolerance target.

**Final challenge:** Design the exact sequence of steps your object store would take to detect and repair a single silently-corrupted fragment of an erasure-coded object — one that reads back successfully (no I/O error) but whose bytes have quietly changed due to disk-level corruption — before a user ever notices. What has to be stored alongside the data to make corruption detectable at all, and how does the system decide it's safe to trust the "repaired" version afterward?

---

# 69. Final Takeaway

Every layer in this guide answers the same underlying question at a different point: **what does this piece of storage let you address, and what does it guarantee once you've addressed it?** Block storage answers it with the barest possible contract — a numbered slot that reliably holds whatever was last written. File storage builds names, hierarchy, and crash-consistent metadata on top of that contract. Object storage strips the hierarchy back out specifically to make horizontal scale tractable, trading in-place mutation for immutability and versioning. And RAID — first within one machine, then generalized into replication and erasure coding across many — answers a question all three share: what happens when the hardware underneath any of this actually fails. None of these are competing technologies to memorize trivia about; they're the same small set of trade-offs (addressing, metadata, mutability, redundancy) resolved differently for genuinely different requirements — and being able to derive each resolution from first principles, the way this guide built every one of them, is what turns "I know what RAID 5 is" into an actual system-design answer.

GFS/HDFS's centralized-master design (§54–§61) rounds this out with a fourth resolution of the exact same "where is my data" question S3-style consistent hashing already answered differently (§60) — proof that even "how do you locate data across a fleet of machines" has more than one defensible answer, and that the right one depends entirely on whether your workload looks like billions of small, independent objects or a smaller number of enormous, sequentially-read files.


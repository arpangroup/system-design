# Design a Version Control System Like Git — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a distributed version control system like Git. Cover both high-level design (the object model, storage, distribution) and low-level design (class diagrams, core algorithms like diff, merge, and garbage collection). Follow SOLID design principles, and be ready to justify every decision when I push back."*
>
> This guide is structured exactly as that interview unfolds: a requirements-gathering phase, a high-level architecture built up decision by decision, a low-level class design deep dive, and a sequence of escalating **follow-up questions** — each answered with real reasoning and, where it matters, real Java code — not a single static diagram presented as if no one ever questioned it.

---

# 1. What We Are Building

We are building **MiniGit** — a distributed version control system covering:

- **Functional requirements**: content-addressable storage of file snapshots, directory-tree snapshots, commits forming a history graph, branches and tags, a staging area, diffing, merging (including conflicts), rebasing, and remote push/pull.
- **High-level design**: why Git is *distributed* rather than client-server, the three-tree architecture (working directory / staging area / repository), and how every clone ends up a complete, independent copy of the entire history.
- **Low-level design**: a real content-addressable object store, a Composite-pattern tree model, a real longest-common-subsequence diff algorithm, a real lowest-common-ancestor merge-base algorithm, and a real three-way merge with conflict detection.
- **Scaling concerns**: delta compression and packfiles so history doesn't grow linearly with every full snapshot, mark-and-sweep garbage collection for unreachable objects, and making history traversal fast on a repository with millions of commits.

```text
   Working Directory  --(git add)-->  Staging Area (Index)  --(git commit)-->  Repository (Object Store + Refs)
   (files you're editing)              (what will be in                        (immutable history: blobs,
                                         the NEXT commit)                        trees, commits, all content-
                                                                                  addressed by hash)

   Every clone contains a COMPLETE, independent copy of the Repository above --
   there is no central server anything is required to talk to.
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain, precisely, why Git is distributed rather than client-server, and what specific capabilities that buys (and what it costs).
- Design a content-addressable object store where identical content is automatically deduplicated, and explain why that alone makes the whole system tamper-evident.
- Model a directory snapshot as a Composite of blobs and sub-trees, and a commit as an immutable node in a directed acyclic graph of history.
- Implement a real diff algorithm (longest common subsequence), a real merge-base algorithm (lowest common ancestor in a DAG), and a real three-way merge with conflict detection — not hand-wave past the parts that are actually hard.
- Explain exactly what a "branch" is at the storage level (a pointer, nothing more), and why that fact is what makes creating one instant and cheap.
- Reason about what breaks first as a repository's history grows into the millions of commits, and name the specific techniques (delta compression, packfiles, garbage collection, commit-graph caching) that address each bottleneck.

---

# 3. Why This Matters (The Interview, Framed)

"Design a version control system like Git" is a favorite senior/staff systems-design question because almost every candidate has *used* Git for years without ever having to explain *how it works internally* — which makes it an unusually good filter for the difference between familiarity and understanding. The question moves across the same three levels every serious design interview probes:

- **Requirements-driven scoping** — "version control system" could mean a centralized model (SVN/CVS) or a distributed one (Git/Mercurial); a strong candidate states which one, and why, before designing anything, the same discipline [the Workflow Automation guide's opening](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) and [the API Gateway guide's opening](<Design an API Gateway and a Rate Limiter — HLD, LLD, and Class Design From Scratch.md>) both insist on.
- **High-level architecture** — the object model, the three-tree working model, and the distribution story, each with real reasoning behind it.
- **Low-level, algorithmic design** — this is where the question gets genuinely hard: a candidate who can describe "commits form a tree" fluently often cannot actually implement a merge-base algorithm, a real diff, or explain why a branch costs nothing to create. That gap is exactly what this guide closes, with real code for every one of those pieces.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and algorithm implementations. |
| Object hashing | SHA-256 | Content-addressing needs a cryptographic hash where a collision is computationally infeasible — the same property [the TinyDB Storage Engine guide's CRC32 checksums](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) deliberately do NOT provide (CRC32 detects accidental corruption; it is not collision-resistant against a crafted input), which is exactly why Git's object store needs a stronger hash than a page checksum does (§15). |
| Object storage | A local, content-addressed filesystem directory, one file per object | The simplest correct implementation of "store bytes, keyed by their own hash" (§16) — the same content-addressable idea [the File Storage guide's object storage section](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) already introduces, applied here to version history instead of arbitrary blobs. |
| Compression | A general-purpose byte-level compressor (zlib-style) plus custom delta encoding | Individual objects are compressed; packfiles (§49) additionally store *differences* between similar objects, not full copies. |
| Diff algorithm | Longest Common Subsequence (Myers-style) | The real algorithm every practical text diff tool is built on (§31-§32). |

---

# 5. Project Structure

```text
minigit/
├── src/main/java/com/example/minigit/
│   ├── objects/
│   │   ├── GitObject.java (sealed), Blob.java, Tree.java, Commit.java, Tag.java   // §16-§22
│   │   └── ObjectStore.java                                                        // §16
│   ├── refs/
│   │   ├── Ref.java, RefStore.java                                                 // §25
│   │   └── Head.java                                                               // §25
│   ├── index/
│   │   └── Index.java, IndexEntry.java                                             // §29
│   ├── diff/
│   │   ├── DiffAlgorithm.java, MyersDiff.java                                      // §32
│   │   └── DiffLine.java (sealed)                                                  // §32
│   ├── merge/
│   │   ├── MergeBaseFinder.java                                                    // §35
│   │   ├── ThreeWayMerge.java, MergeConflict.java                                  // §37
│   │   └── Rebaser.java                                                            // §42
│   ├── remote/
│   │   ├── RemoteRepository.java                                                   // §44
│   │   └── PushNegotiator.java                                                     // §45-§46
│   ├── pack/
│   │   └── DeltaEncoder.java, PackFile.java                                        // §49-§50
│   └── gc/
│       └── GarbageCollector.java, ReachabilityWalker.java                          // §52-§53
└── src/test/java/com/example/minigit/
    ├── ObjectStoreDeduplicationTest.java
    ├── ThreeWayMergeConflictTest.java
    └── GarbageCollectionReachabilityTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"Distributed, like Git, or centralized, like SVN? Do we need to support binary files efficiently, or is this scoped to text source code? What's the expected scale — thousands of commits, or millions, across how many contributors? Do we need a network protocol for push/pull in real depth, or is local repository design the focus?"*

Exactly as [the API Gateway guide's opening](<Design an API Gateway and a Rate Limiter — HLD, LLD, and Class Design From Scratch.md>) argues, narrowing an intentionally broad prompt before designing anything is the first move that separates a strong answer from a shallow one. For this guide, we settle on a concrete, realistic scope: **distributed** (every clone is a full repository, §13), **text-focused diffing and merging** (binary files are stored but not line-diffed — a real, standard, stated limitation, §62), targeting histories that grow into **millions of commits** across **thousands of contributors**, with **real push/pull negotiation logic** covered in genuine depth (§43-§46), not hand-waved.

---

# 7. Functional Requirements

- Take a **snapshot** of the working directory's current state and store it, content-addressed, so identical content is never stored twice.
- Represent a **directory structure** (files and subdirectories) as a first-class, hashable object.
- Record a **commit**: a snapshot plus a message, author, timestamp, and a pointer to its parent commit(s).
- Support **branches** — independent, cheaply-created lines of development — and **tags** — a fixed, named pointer to one specific commit.
- Maintain a **staging area**, so a commit's contents are chosen deliberately, not implicitly "everything currently in the working directory."
- Compute a **diff** between any two snapshots, at the line level, for a human to read.
- **Merge** two diverged branches, automatically where possible, and clearly flag a **conflict** where not.
- **Rebase** a branch's commits onto a new base.
- **Push** and **pull** commits to and from a remote repository, transferring only the objects the other side doesn't already have.
- Reclaim storage for commits that are no longer reachable from any branch or tag (**garbage collection**).

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Storage efficiency** | History should not grow linearly with "one full copy of every file, per commit, forever" | Content-addressable deduplication (§15-§16), delta compression and packfiles (§48-§50) |
| **Integrity** | It must be computationally infeasible to silently corrupt or forge history | SHA-256 content-addressing (§15) — every object's identity IS a cryptographic commitment to its own bytes |
| **Distributed, offline-first operation** | Every operation except push/pull must work with zero network access | The full-clone model (§13) |
| **Correctness of merges** | A merge must correctly identify true conflicts, and never silently drop a change | The three-way merge algorithm (§36-§37) |
| **Performance at scale** | Common operations must stay fast as history grows into millions of commits | Commit-graph caching, bitmap indexes (§55) |
| **Extensibility** | Adding a new merge strategy or diff algorithm must not require rewriting the core object model | The Strategy pattern applied to both (§58) |

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Before architecture — what actually exists in this system? Not commands. Nouns."*

This is the same deliberate pivot [the Workflow Automation guide's own Follow-up 1](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) and [the API Gateway guide's Follow-up 1](<Design an API Gateway and a Rate Limiter — HLD, LLD, and Class Design From Scratch.md>) both make — naming the domain model before naming components keeps the design honest about what actually needs solving.

---

# 10. Identifying the Core Domain Entities

| Entity | Represents | Key relationships |
|---|---|---|
| **Blob** | The raw content of one file, at one point in time | Identified purely by the hash of its own bytes (§15-§16) |
| **Tree** | A directory snapshot — a list of named entries, each pointing to a Blob or another Tree | Composite structure (§18-§19) |
| **Commit** | One point in history: a Tree snapshot, a message, an author, a timestamp, and parent commit(s) | Forms a DAG via its parent pointers (§21) |
| **Ref** | A named, mutable pointer to a commit hash — a branch or a tag | The *only* mutable state in the whole model (§24-§25) |
| **HEAD** | A pointer to "the ref (or commit) currently checked out" | Determines what a new commit's parent will be |
| **Index** | The staging area — what will be in the *next* commit | Sits between the working directory and the object store (§28-§29) |
| **ObjectStore** | The content-addressable storage for every Blob/Tree/Commit ever created | Append-only, immutable, deduplicating (§16) |

Every section from §11 onward either builds infrastructure **around** these entities or builds the entities **themselves** as real, working class designs — nothing introduced later is untraceable back to this table.

---

# 11. High-Level Architecture Overview

```text
                          Working Directory
                       (files you're actively editing)
                                  |
                          git add (stages changes)
                                  v
                          Index / Staging Area
                       (exactly what the NEXT commit will contain)
                                  |
                          git commit (snapshots the index)
                                  v
                    Repository = ObjectStore + Refs + HEAD
              +-----------------------------------------------+
              |  ObjectStore: content-addressed Blobs/Trees/   |
              |  Commits, immutable, deduplicated (§15-§22)    |
              |  Refs: branches/tags -- mutable pointers (§24) |
              +-----------------------------------------------+
                                  |
                       git push / git pull / git fetch
                                  v
                       Remote Repository (structurally
                       IDENTICAL to a local one, §13)
```

The single most important property this diagram is building toward: a "remote" repository is not a fundamentally different kind of thing from a local one — it's the *exact same* `ObjectStore` + `Refs` structure, reachable over a network instead of a local disk. §13 makes this precise.

---

# 12. Follow-up Question 2 — "Why Is This Distributed, Not Client-Server Like SVN? What Does That Actually Change?"

> **Interviewer:** *"SVN has one central server holding history; clients check out a working copy. What does making this distributed instead actually buy you, concretely — not as a buzzword?"*

Three concrete, specific capabilities a client-server model can't offer, each worth naming separately: **every operation that doesn't involve syncing with someone else — committing, branching, viewing history, diffing — works completely offline**, because the full history is already local; **there is no single point of failure for the history itself** — losing the "central" server loses nothing that any one clone doesn't already have a complete copy of; and **any two people can share history directly with each other**, with no requirement that either of them has write access to some designated "the" server. §13 makes precise exactly what "every clone is a full repository" means at the data-structure level.

---

# 13. The Distributed Model: Every Clone Is a Full Repository

Cloning a repository means copying its entire `ObjectStore` (every Blob, Tree, and Commit ever created) and its `Refs` — not a lightweight reference to a remote, and not a partial checkout of just the current state. This has a direct, load-bearing consequence for everything else in this guide: **every algorithm this guide builds — diffing, merging, finding a merge base, walking history — operates purely on LOCAL data**, with no network call anywhere inside any of them. The network only ever enters the picture at exactly one seam, push/pull (§43-§46), which is deliberately the *only* place in the entire design where two independent object stores need to reconcile with each other.

---

# 14. Follow-up Question 3 — "How Do You Store a File's Content So Identical Content Is Never Stored Twice?"

> **Interviewer:** *"Two commits both contain a file with byte-for-byte identical content — maybe it's a config file nobody's touched in months. How do you avoid storing that content twice, without maintaining some separate 'have I seen this before' index?"*

§15-§16 answer this with the single idea almost everything else in this guide's object model builds on: **content-addressable storage** — an object's storage key *is* the cryptographic hash of its own content, so two objects with identical bytes are, by construction, the exact same stored object.

---

# 15. Content-Addressable Storage: Hashing Content Into Object IDs

An object's identity is `SHA-256(content)` — never an incrementing counter, never a filename, never anything assigned externally. This single decision has three consequences worth stating explicitly:

- **Automatic deduplication**: storing the same bytes twice produces the same hash, so a second `store(bytes)` call for content already present is a no-op by construction — no separate "check if this exists" index is ever needed, the hash function itself *is* the check.
- **Tamper-evidence**: an object's hash is a cryptographic commitment to its exact bytes — flip a single bit anywhere in an object's content, and its hash changes completely, which (once §21's commits chain hashes together) makes silently altering history without detection computationally infeasible.
- **A stronger hash than a corruption check needs**: this is deliberately SHA-256, not something like CRC32. The [TinyDB Storage Engine guide's page checksums](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) use CRC32 specifically because they only need to detect *accidental* corruption (a bit flip from a failing disk) — CRC32 is fast and that's sufficient for that job. Here, the hash *is* the object's identity and address, and it must also resist a *deliberate* attempt to construct two different pieces of content sharing one hash — a fundamentally different, stronger requirement.

---

# 16. Implementing the Blob Object and the Object Store

```java
public sealed interface GitObject permits Blob, Tree, Commit, Tag {
    byte[] serialize();
}
```

```java
public record Blob(byte[] content) implements GitObject {
    @Override
    public byte[] serialize() { return content; } // a blob's serialized form IS just its raw bytes
}
```

```java
public final class ObjectStore {

    private final Path objectsDirectory;

    public ObjectStore(Path objectsDirectory) { this.objectsDirectory = objectsDirectory; }

    public String store(GitObject object) throws IOException {
        byte[] serialized = object.serialize();
        String hash = sha256Hex(serialized);
        Path objectPath = pathFor(hash);
        if (!Files.exists(objectPath)) { // the ENTIRE deduplication check -- nothing more is needed
            Files.createDirectories(objectPath.getParent());
            Files.write(objectPath, compress(serialized));
        }
        return hash;
    }

    public byte[] read(String hash) throws IOException {
        return decompress(Files.readAllBytes(pathFor(hash)));
    }

    private Path pathFor(String hash) {
        // Git's own convention: first 2 hex chars as a subdirectory, avoiding millions of files in one directory
        return objectsDirectory.resolve(hash.substring(0, 2)).resolve(hash.substring(2));
    }

    private String sha256Hex(byte[] bytes) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256").digest(bytes);
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 must always be available", e);
        }
    }

    // compress(...)/decompress(...) -- a standard byte-level compressor (zlib-style), elided here; §49 layers
    // delta compression ON TOP of this per-object compression for objects similar to one already stored.
}
```

`if (!Files.exists(objectPath))` before writing is the entire deduplication mechanism — there is no separate index, no lookup table, nothing to keep in sync, because the hash computed from the content *is* the lookup key, and two calls with identical content always compute the identical key.

---

# 17. Follow-up Question 4 — "A Commit Needs a Directory Structure, Not Just Files. How Do You Represent That?"

> **Interviewer:** *"A real project has files nested in directories. A single Blob is just one file's bytes — how do you snapshot an entire directory tree?"*

§18-§19 answer with a second object type, `Tree`, that can point to Blobs *and* to other Trees — a directory listing, made hashable and content-addressed exactly like a Blob.

---

# 18. The Tree Object: Modeling Directory Structure

```text
Tree "src/"
├── entry: "main.py"     -> Blob   b4f2...  (a file)
├── entry: "utils.py"    -> Blob   9ac1...  (a file)
└── entry: "helpers/"    -> Tree   7e3d...  (a SUBDIRECTORY -- another Tree, recursively)
```

A `Tree` never stores a file's content directly — every entry is a *name plus a hash*, pointing at either a `Blob` (a file) or another `Tree` (a subdirectory). This is a direct application of the **Composite** pattern: `Tree` and `Blob` are both addressable by hash, and a `Tree` can contain either kind of child uniformly, which is exactly what makes recursively hashing an entire directory structure — file, or a directory of files, or a directory of directories of files — the same operation applied at every level.

---

# 19. Implementing the Tree Object

```java
public record TreeEntry(String name, String mode /* "100644" file, "040000" directory */, String hash) { }

public record Tree(List<TreeEntry> entries) implements GitObject {
    @Override
    public byte[] serialize() {
        // Entries sorted by name FIRST -- this is not cosmetic: it guarantees two Trees with the identical
        // set of entries always serialize to identical bytes, and therefore hash to the identical object,
        // regardless of the order entries happened to be added in. Without this, the SAME directory
        // snapshot could hash differently depending on filesystem iteration order -- breaking deduplication.
        List<TreeEntry> sorted = entries.stream().sorted(Comparator.comparing(TreeEntry::name)).toList();
        StringBuilder sb = new StringBuilder();
        for (TreeEntry entry : sorted) {
            sb.append(entry.mode()).append(' ').append(entry.name()).append('\0').append(entry.hash()).append('\n');
        }
        return sb.toString().getBytes(StandardCharsets.UTF_8);
    }
}
```

```java
public final class TreeBuilder {

    private final ObjectStore objectStore;

    public TreeBuilder(ObjectStore objectStore) { this.objectStore = objectStore; }

    /** Recursively snapshots a directory into a Tree object, storing every Blob and Tree it touches. */
    public String snapshotDirectory(Path directory) throws IOException {
        List<TreeEntry> entries = new ArrayList<>();
        for (Path child : listChildren(directory)) {
            if (Files.isDirectory(child)) {
                String subtreeHash = snapshotDirectory(child); // recursive -- a Tree's children can be Trees
                entries.add(new TreeEntry(child.getFileName().toString(), "040000", subtreeHash));
            } else {
                String blobHash = objectStore.store(new Blob(Files.readAllBytes(child)));
                entries.add(new TreeEntry(child.getFileName().toString(), "100644", blobHash));
            }
        }
        return objectStore.store(new Tree(entries));
    }
}
```

The sorted-serialization detail in `Tree.serialize()` is the kind of correctness detail that's invisible until it silently breaks deduplication — two functionally identical directory snapshots, built by code that happened to iterate their entries in a different order, must still hash to the *same* Tree object, and deterministic sorting is what guarantees that.

---

# 20. Follow-up Question 5 — "How Do You Tie a Tree Snapshot to a Point in Project History?"

> **Interviewer:** *"A Tree is a directory snapshot at ONE instant. History is a sequence of these, with authorship and messages attached. How do you model that?"*

§21-§22 add the third object type, `Commit` — a Tree, plus the metadata a snapshot needs to become a meaningful point in *history*, plus a pointer to whatever came immediately before it.

---

# 21. The Commit Object: Snapshots, Parents, and the DAG

```text
Commit c3 (parents: [c2])          <-- a normal commit, one parent
   |
Commit c2 (parents: [c1])
   |
Commit c1 (parents: [])            <-- the root commit, no parent at all
```

```text
Commit m (parents: [c2, b2])       <-- a MERGE commit -- TWO parents, one from each branch being merged
   |          \
Commit c2      Commit b2
   |              |
Commit c1 ------- (both branches share this common ancestor)
```

A commit's `parents` list is what turns a flat sequence of snapshots into a **directed acyclic graph**: a normal commit has exactly one parent, a merge commit has two (or, rarely, more), and the root commit of a repository has none at all. This DAG can **never** contain a cycle, for a reason worth stating precisely: a commit's hash is computed *from* its parent hashes (§22), so a commit literally cannot exist, cannot be created, cannot be assigned a hash, until every one of its parents already exists and already has a hash — a cycle would require a commit to be its own ancestor, which would require it to exist before it exists. This is a fundamentally different, and stronger, guarantee than [the Workflow Automation guide's `WorkflowValidator`](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) achieves by *validating* a workflow definition's graph for cycles after the fact — here, the hash-chained construction makes a cycle structurally impossible to create in the first place, not merely checked-for-and-rejected.

---

# 22. Implementing the Commit Object

```java
public record Commit(String treeHash, List<String> parentHashes, String author, String message, long timestampEpochSeconds)
        implements GitObject {
    @Override
    public byte[] serialize() {
        StringBuilder sb = new StringBuilder();
        sb.append("tree ").append(treeHash()).append('\n');
        for (String parentHash : parentHashes()) { // zero, one, or (for a merge) two-or-more entries
            sb.append("parent ").append(parentHash).append('\n');
        }
        sb.append("author ").append(author()).append(' ').append(timestampEpochSeconds()).append('\n');
        sb.append('\n').append(message());
        return sb.toString().getBytes(StandardCharsets.UTF_8);
    }
}
```

```java
public final class CommitBuilder {

    private final ObjectStore objectStore;

    public CommitBuilder(ObjectStore objectStore) { this.objectStore = objectStore; }

    public String commit(String treeHash, List<String> parentHashes, String author, String message) throws IOException {
        Commit commit = new Commit(treeHash, parentHashes, author, message, Instant.now().getEpochSecond());
        return objectStore.store(commit); // the commit's hash depends on parentHashes -- see §21's cycle-impossibility argument
    }
}
```

---

# 23. Follow-up Question 6 — "What's a Branch, Actually? Walk Me Through What Changes When I Create One."

> **Interviewer:** *"Creating a branch on a repository with 500,000 commits is instant. It doesn't copy anything. What is a branch, structurally, that makes that true?"*

§24-§25 answer with the single fact that makes the "instant, cheap branch" property true by construction: a branch **is** a pointer, and nothing more.

---

# 24. Branches Are Just Pointers: The Ref Model

A branch named `feature-x` is a file (or a database row, or a key in any simple store) containing exactly one thing: **the hash of the commit it currently points to.** Creating a branch is writing one new pointer, copying the *current* commit hash into it — it never copies a single commit, a single tree, or a single blob, because none of that data belongs to the branch; it all already lives in the shared `ObjectStore` (§16), addressed by hash, completely independent of how many refs happen to point at it. A commit made *on* `feature-x` doesn't append to some branch-owned list — it creates a brand new `Commit` object (§22) whose parent is the commit `feature-x` currently points to, and then **moves** the `feature-x` pointer to the new commit's hash. Deleting a branch is deleting that one pointer — the commits it pointed to don't disappear (they may still be reachable from another ref), and if they aren't reachable from anything, §52's garbage collector is what eventually reclaims them, not branch deletion itself.

---

# 25. Implementing Refs and HEAD

```java
public final class RefStore {

    private final Path refsDirectory;

    public RefStore(Path refsDirectory) { this.refsDirectory = refsDirectory; }

    public void setRef(String refName, String commitHash) throws IOException {
        Path refPath = refsDirectory.resolve(refName);
        Files.createDirectories(refPath.getParent());
        Files.writeString(refPath, commitHash); // ONE pointer write -- §24's entire "branch creation" cost
    }

    public Optional<String> resolveRef(String refName) throws IOException {
        Path refPath = refsDirectory.resolve(refName);
        return Files.exists(refPath) ? Optional.of(Files.readString(refPath).strip()) : Optional.empty();
    }

    public void deleteRef(String refName) throws IOException {
        Files.deleteIfExists(refsDirectory.resolve(refName)); // deletes the POINTER ONLY -- never touches the object store
    }
}
```

```java
public final class Head {

    private final Path headFile;

    public Head(Path headFile) { this.headFile = headFile; }

    /** HEAD normally points at a REF NAME (e.g. "refs/heads/main"), not a raw commit hash -- this is what
     *  makes committing on a branch automatically move that branch's pointer forward, §24. */
    public void pointToRef(String refName) throws IOException { Files.writeString(headFile, "ref: " + refName); }

    /** "Detached HEAD" -- points directly at a commit hash, bypassing any branch. A new commit here has
     *  nowhere automatic to update its parent branch to, which is exactly why Git warns about this state. */
    public void pointToCommit(String commitHash) throws IOException { Files.writeString(headFile, commitHash); }

    public boolean isDetached() throws IOException {
        return !Files.readString(headFile).strip().startsWith("ref: ");
    }
}
```

`Head.isDetached()` existing at all is a direct, necessary consequence of §24's model: since a branch is *just* a pointer, it's entirely possible to check out a raw commit hash with no branch pointing at it — a state real Git surfaces explicitly ("detached HEAD") precisely because a commit made there has no ref that will automatically follow it, and it can become unreachable and garbage-collected (§52) the moment `HEAD` moves elsewhere, unless the user deliberately creates a branch there first.

---

# 26. Class Diagram: The Object Model and Refs

```text
GitObject <<sealed interface>>                       RefStore
+ serialize(): byte[]                                + setRef(name, commitHash)
      ^                                                + resolveRef(name): commitHash
      | permits                                       + deleteRef(name)
  +---+------+--------+                                      ^
 Blob      Tree     Commit   Tag                              | reads/writes
  |          |         |                                      |
  | content  | entries | treeHash, parentHashes[]        Head
  |          |    |    | author, message, timestamp      + pointToRef(name)
  |          v    |                                       + pointToCommit(hash)
  |      TreeEntry|                                       + isDetached(): boolean
  |      + name, mode, hash
  |          (points to Blob OR Tree -- Composite)
  v
ObjectStore
+ store(object): hash        <-- content-addressed, §15
+ read(hash): bytes           deduplicating by construction
```

Every arrow into `ObjectStore` is one-directional and by-hash — nothing in this diagram ever holds a direct in-memory reference to another object; every relationship (a Tree's entry, a Commit's parent, a Ref's target) is a **hash string**, resolved through `ObjectStore.read()` only when something actually needs to look at the referenced object's contents. This is precisely what makes the whole structure trivially shareable across a network (§43-§46) — a hash means the exact same thing in every clone of the repository.

---

# 27. Follow-up Question 7 — "There's a Staging Area Between My Working Directory and a Commit. Why Does That Layer Exist?"

> **Interviewer:** *"You could just commit whatever's currently in the working directory. Why does Git insist on an explicit staging step in between?"*

Because "everything currently in the working directory" and "what I actually want in the next commit" are frequently *not* the same set of changes — a developer routinely has several unrelated edits sitting in their working directory at once, and wants to commit only some of them, as one deliberate, coherent snapshot. §28-§29 build the layer that makes that possible.

---

# 28. The Three-Tree Architecture: Working Directory, Index, Repository

```text
Working Directory              Index (Staging Area)              Repository
(the actual files on disk,     (a snapshot of exactly what        (immutable history --
 possibly mid-edit)             the NEXT commit will contain)      every past commit)
       |                               |                                 |
       |  git add <file>               |                                 |
       +------------------------------>|                                 |
       |                               |  git commit                     |
       |                               +-------------------------------->|
       |  git checkout                 |                                 |
       |<---------------------------------------------------------------+
```

Three genuinely distinct states, each answering a different question: the working directory answers *"what does the developer currently see?"*; the index answers *"what will the next commit contain, exactly?"*; the repository answers *"what has already been permanently recorded?"* Collapsing the middle layer — committing directly from the working directory — would make "commit only some of my changes" impossible to express at all.

---

# 29. Implementing the Index (Staging Area)

```java
public record IndexEntry(String path, String blobHash, String mode) { }

public final class Index {

    private final Path indexFile;
    private Map<String, IndexEntry> entriesByPath;

    public Index(Path indexFile) throws IOException {
        this.indexFile = indexFile;
        this.entriesByPath = load();
    }

    public void stage(String path, String blobHash, String mode) throws IOException {
        entriesByPath.put(path, new IndexEntry(path, blobHash, mode));
        persist();
    }

    public void unstage(String path) throws IOException {
        entriesByPath.remove(path);
        persist();
    }

    /** Builds the Tree that will become the next commit's snapshot -- ONLY from staged entries, never
     *  from whatever else happens to be sitting in the working directory. */
    public String buildTreeFromStagedEntries(ObjectStore objectStore) throws IOException {
        List<TreeEntry> treeEntries = entriesByPath.values().stream()
                .map(e -> new TreeEntry(e.path(), e.mode(), e.blobHash()))
                .toList();
        return objectStore.store(new Tree(treeEntries)); // §19 -- deterministic sort happens inside Tree.serialize()
    }

    private Map<String, IndexEntry> load() throws IOException { /* deserialize indexFile, or empty if absent */ return new LinkedHashMap<>(); }
    private void persist() throws IOException { /* serialize entriesByPath back to indexFile */ }
}
```

`git add` calls `stage(...)`; `git commit` calls `buildTreeFromStagedEntries(...)` and then §22's `CommitBuilder.commit(...)` with the resulting tree hash — the index is the *only* thing `git commit` ever reads from, which is the concrete mechanism behind "commit only what I explicitly staged."

---

# 30. Follow-up Question 8 — "How Do You Compute What Changed Between Two Versions of a File?"

> **Interviewer:** *"Two Blobs, old content and new content. `git diff` shows exactly which lines changed. How does that algorithm actually work?"*

This is a genuinely hard, classic computer-science problem, not a string-comparison trick — §31 states it precisely; §32 implements the real algorithm.

---

# 31. The Diff Problem: Longest Common Subsequence

The right way to think about "what changed between two versions of a file" is: find the **longest common subsequence (LCS)** of lines between the old version and the new version — the longest sequence of lines that appears, in the same relative order, in both files (not necessarily contiguous). Every line in the old file that is **not** part of that common subsequence was *deleted*; every line in the new file that is **not** part of it was *added*. This reframing is what turns "diff" from a vague notion into a well-defined, solvable optimization problem — specifically, one solvable with dynamic programming, the same category of technique the [TinyDB Query Engine guide's cost-based access-path selection](<TinyDB Query Engine and SQL Parser — Step-by-Step Implementation Guide.md>) uses for a different optimization problem entirely.

---

# 32. Implementing a Real Line-Based Diff Algorithm

```java
public sealed interface DiffLine permits Unchanged, Added, Removed {
    String text();
}
public record Unchanged(String text) implements DiffLine { }
public record Added(String text) implements DiffLine { }
public record Removed(String text) implements DiffLine { }
```

```java
public final class MyersDiff implements DiffAlgorithm {

    @Override
    public List<DiffLine> diff(List<String> oldLines, List<String> newLines) {
        int m = oldLines.size(), n = newLines.size();
        int[][] lcsLength = new int[m + 1][n + 1]; // lcsLength[i][j] = length of the LCS of oldLines[0..i), newLines[0..j)

        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                if (oldLines.get(i - 1).equals(newLines.get(j - 1))) {
                    lcsLength[i][j] = lcsLength[i - 1][j - 1] + 1;
                } else {
                    lcsLength[i][j] = Math.max(lcsLength[i - 1][j], lcsLength[i][j - 1]);
                }
            }
        }
        return backtrack(oldLines, newLines, lcsLength, m, n);
    }

    /** Walks the DP table backward from (m,n) to (0,0), emitting one DiffLine per step -- this is what
     *  turns the LCS LENGTH (just a number) into an actual, ordered list of unchanged/added/removed lines. */
    private List<DiffLine> backtrack(List<String> oldLines, List<String> newLines, int[][] lcsLength, int i, int j) {
        Deque<DiffLine> result = new ArrayDeque<>();
        while (i > 0 || j > 0) {
            if (i > 0 && j > 0 && oldLines.get(i - 1).equals(newLines.get(j - 1))) {
                result.addFirst(new Unchanged(oldLines.get(i - 1)));
                i--; j--;
            } else if (j > 0 && (i == 0 || lcsLength[i][j - 1] >= lcsLength[i - 1][j])) {
                result.addFirst(new Added(newLines.get(j - 1)));
                j--;
            } else {
                result.addFirst(new Removed(oldLines.get(i - 1)));
                i--;
            }
        }
        return new ArrayList<>(result);
    }
}
```

The DP table is exactly `O(m*n)` in time and space — perfectly fine for diffing individual files, and precisely why real Git additionally uses heuristics (a sliding window, hashing long runs of identical lines) to stay fast on very large files, a refinement this guide names honestly as future work (§62) rather than building out in full. The **backtrack** step is what most naive descriptions of "LCS-based diff" skip entirely, and it's the actual mechanism that produces the ordered add/remove/unchanged sequence a human reads as a diff — the DP table alone only ever tells you the *length* of the longest common subsequence, never *which* lines it consists of.

---

# 33. Follow-up Question 9 — "Two Branches Diverged and Both Changed the Same File. How Do You Merge Them?"

> **Interviewer:** *"`feature-x` and `main` both branched off the same commit, and both independently modified the same file, differently. `git merge` mostly just works. Walk me through the actual algorithm."*

Every real merge needs three inputs, not two — the two diverged tips, **and** the commit they both diverged *from*. §34-§35 find that third input; §36-§37 use it to actually merge.

---

# 34. Finding the Merge Base: Lowest Common Ancestor in a DAG

The commit both branches diverged from is their **lowest common ancestor (LCA)** in the commit DAG (§21) — the deepest commit that is an ancestor of both tips. Finding it correctly, on a DAG where a commit can have multiple parents (merge commits) and where the two branches' histories can be wildly different lengths, is a genuine graph algorithm, not a shortcut.

```text
        c1
       /  \
     c2    b2
      |     |
     c3    b3   <-- feature-x tip (b3) and main tip (c3) both descend from c1
                     c1 is their lowest common ancestor -- the merge base
```

---

# 35. Implementing LCA

```java
public final class MergeBaseFinder {

    private final ObjectStore objectStore;

    public MergeBaseFinder(ObjectStore objectStore) { this.objectStore = objectStore; }

    public String findMergeBase(String commitA, String commitB) throws IOException {
        Set<String> ancestorsOfA = ancestorsIncludingSelf(commitA);

        // BFS outward from commitB -- the FIRST commit encountered that's also an ancestor of A
        // is guaranteed to be the LOWEST (deepest/most-recent) common ancestor, never a shallower one,
        // precisely because BFS visits commits in non-decreasing distance from commitB.
        Queue<String> queue = new ArrayDeque<>(List.of(commitB));
        Set<String> visited = new HashSet<>();
        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (!visited.add(current)) continue;
            if (ancestorsOfA.contains(current)) return current;
            for (String parent : parentsOf(current)) queue.add(parent);
        }
        throw new NoCommonAncestorException(commitA, commitB); // two genuinely unrelated histories
    }

    private Set<String> ancestorsIncludingSelf(String commitHash) throws IOException {
        Set<String> ancestors = new HashSet<>();
        Deque<String> stack = new ArrayDeque<>(List.of(commitHash));
        while (!stack.isEmpty()) {
            String current = stack.pop();
            if (!ancestors.add(current)) continue;
            for (String parent : parentsOf(current)) stack.push(parent);
        }
        return ancestors;
    }

    private List<String> parentsOf(String commitHash) throws IOException {
        Commit commit = (Commit) GitObjectParser.parse(objectStore.read(commitHash));
        return commit.parentHashes();
    }
}
```

Computing the *full* ancestor set of `commitA` first, then walking outward from `commitB` breadth-first and stopping at the first hit, is what guarantees correctness on a real DAG with merge commits — a naive "walk both branches' first-parent chains and compare" approach silently gives the wrong answer the moment either branch's history includes its own earlier merge, which any real-world repository's history does constantly.

---

# 36. The Three-Way Merge Algorithm

With the merge base (§34-§35) in hand, a merge has exactly three trees to compare, per file: the **base** version (before either branch touched it), branch **A**'s version, and branch **B**'s version. Per line, or per hunk of lines, the decision is mechanical:

```text
base == A, base != B   -> take B's change (only B touched this)
base == B, base != A   -> take A's change (only A touched this)
base == A == B         -> unchanged, nothing to do
A == B, both != base    -> both branches made the SAME change independently -- take it, no conflict
base != A, base != B, A != B  -> BOTH branches changed this differently -- a genuine CONFLICT
```

The last case — both sides changed the *same* region *differently* — is the only one requiring a human, and is precisely why a three-way merge needs the base at all: without it, there would be no way to distinguish "branch A added this line" from "branch A and branch B both happened to write the same content," which a plain two-way diff (§31-§32) between A and B alone cannot tell apart.

---

# 37. Implementing a Three-Way Merge

```java
public record MergeConflict(String path, List<String> baseLines, List<String> oursLines, List<String> theirsLines) { }

public final class ThreeWayMerge {

    private final DiffAlgorithm diffAlgorithm; // §32's MyersDiff

    public ThreeWayMerge(DiffAlgorithm diffAlgorithm) { this.diffAlgorithm = diffAlgorithm; }

    public MergeResult merge(List<String> base, List<String> ours, List<String> theirs) {
        List<DiffLine> baseToOurs = diffAlgorithm.diff(base, ours);
        List<DiffLine> baseToTheirs = diffAlgorithm.diff(base, theirs);

        boolean oursChangedFromBase = baseToOurs.stream().anyMatch(l -> !(l instanceof Unchanged));
        boolean theirsChangedFromBase = baseToTheirs.stream().anyMatch(l -> !(l instanceof Unchanged));

        if (!oursChangedFromBase) return MergeResult.clean(theirs);   // only theirs changed -- take theirs
        if (!theirsChangedFromBase) return MergeResult.clean(ours);   // only ours changed -- take ours
        if (ours.equals(theirs)) return MergeResult.clean(ours);      // both changed IDENTICALLY -- no conflict

        return MergeResult.conflict(new MergeConflict("<path>", base, ours, theirs)); // §36's genuine conflict case
    }
}
```

This deliberately simplified, whole-file-granularity version captures the real decision structure (§36's five cases) without the added complexity of a real implementation's line-range/hunk-level conflict detection, which narrows a conflict down to the specific overlapping lines rather than flagging an entire file — a refinement worth naming explicitly (§62) rather than building out here, since the *algorithmic* insight (three inputs, not two; the base is what makes a true conflict distinguishable from a coincidental identical edit) is unchanged either way.

---

# 38. Follow-up Question 10 — "What's a Fast-Forward Merge, and When Does Git Use One Instead of a True Merge Commit?"

> **Interviewer:** *"Sometimes merging a branch doesn't create a merge commit at all — the target branch's pointer just moves forward. When does that happen, and why is it safe?"*

§39 answers with the one specific condition that makes creating an actual merge commit (§21's two-parent case) entirely unnecessary.

---

# 39. Fast-Forward vs. True Merge

A **fast-forward** merge is possible exactly when the merge base (§34-§35) **is** the target branch's current tip — meaning the target branch hasn't moved at all since the two diverged, so there's nothing on the target side to reconcile with. In that case, "merging" the other branch in is nothing more than moving the target's ref (§24-§25) forward to the other branch's tip commit — no new commit is created, because §36's three-way comparison would find that `oursChangedFromBase` is `false` for *every single file*, which §37's first branch already handles by simply taking the other side wholesale. A **true merge**, creating a real two-parent commit, is required exactly when both sides have moved since diverging — the general case §36-§37 were built for. Recognizing the fast-forward case up front is purely an optimization (skip an unnecessary merge commit when the three-way logic would produce a trivial, no-conflict result anyway) — it changes nothing about correctness, only about how much history-graph clutter accumulates from merges that never needed to be genuine merges at all.

---

# 40. Follow-up Question 11 — "Rebase vs. Merge — What's Actually Different Under the Hood?"

> **Interviewer:** *"Both `merge` and `rebase` reconcile a branch with another. What's structurally different about what each one actually does to the commit graph?"*

A merge (§36-§37) **combines** two histories, preserving both, by creating a new commit with two parents — the DAG grows a new node linking two existing branches together, and every commit that ever existed stays exactly where it was. A rebase **rewrites** history — it takes a branch's commits and re-creates them, one by one, as if they had been made starting from a *different* base commit, producing entirely new commit objects (new hashes, since a commit's hash depends on its parent, §21-§22) with the *same* content changes. §41-§42 build this precisely.

---

# 41. Rebasing: Replaying Commits Onto a New Base

```text
Before:  main:      c1 - c2 - c3
         feature:         \ f1 - f2

git rebase main (while on feature)

After:   main:      c1 - c2 - c3
         feature:              \ f1' - f2'   <-- NEW commits, same content changes, different parent/hash
```

Rebasing does not move `f1`/`f2` — it **cannot**, because a commit's identity is its hash, computed from its parent (§21-§22), and `f1`'s parent is changing from `c1`'s old position to `c3`. It computes the diff each original commit introduced relative to *its own* parent, then re-applies that same diff on top of the new base, one commit at a time, creating a brand-new commit object at each step.

---

# 42. Implementing Rebase

```java
public final class Rebaser {

    private final ObjectStore objectStore;
    private final ThreeWayMerge threeWayMerge; // §37 -- reused; replaying a diff onto a new base IS a merge
    private final CommitBuilder commitBuilder; // §22

    public Rebaser(ObjectStore objectStore, ThreeWayMerge threeWayMerge, CommitBuilder commitBuilder) {
        this.objectStore = objectStore;
        this.threeWayMerge = threeWayMerge;
        this.commitBuilder = commitBuilder;
    }

    /** Replays every commit unique to `branchTip` (not already on `newBase`) onto `newBase`, in original order. */
    public String rebase(String branchTip, String oldBase, String newBase) throws IOException {
        List<Commit> commitsToReplay = commitsBetween(oldBase, branchTip); // oldest first
        String currentParent = newBase;

        for (Commit original : commitsToReplay) {
            Commit originalParent = readCommit(original.parentHashes().get(0));
            // The diff THIS commit introduced, relative to ITS OWN original parent -- then merge that
            // same change onto currentParent, exactly the three-way logic §36-§37 already built, just with
            // "the commit's own prior tree" playing the role of the merge base instead of a genuine ancestor.
            MergeResult result = threeWayMerge.merge(
                    linesOf(originalParent.treeHash()), linesOf(currentParent), linesOf(original.treeHash()));

            if (result.hasConflict()) throw new RebaseConflictException(original, result.conflict()); // §36's case 5

            String newTreeHash = objectStore.store(new Tree(treeEntriesFrom(result.mergedLines())));
            currentParent = commitBuilder.commit(newTreeHash, List.of(currentParent), original.author(), original.message());
        }
        return currentParent; // the new tip of the rebased branch
    }

    // commitsBetween(...)/readCommit(...)/linesOf(...)/treeEntriesFrom(...) elided -- straightforward
    // traversal/serialization helpers built from pieces already shown in §16, §19, and §35.
}
```

Reusing `ThreeWayMerge` (§37) for rebase rather than writing a separate "replay a diff" algorithm is deliberate: replaying one commit's change onto a new parent **is** a three-way merge, just with the commit's *own original parent* standing in as the base — the same five-case decision structure (§36) applies unchanged, which is exactly why a rebase can hit a conflict too, for the identical underlying reason a merge can.

---

# 43. Follow-up Question 12 — "How Does the Remote Push/Pull Protocol Actually Work?"

> **Interviewer:** *"§13 said a remote is structurally identical to a local repository. Pushing 3 new commits to a remote with 500,000 existing commits obviously doesn't re-transfer all 500,003. How does it know what to actually send?"*

§44 restates the remote model precisely; §45-§46 build the negotiation that answers "which objects does the other side actually need."

---

# 44. Remote Repositories and Refs

A `RemoteRepository`, from the local repository's point of view, is just another `ObjectStore` + `RefStore` (§16, §25), reachable over a network connection instead of local disk — pushing or pulling is fundamentally **"reconcile two independent hash-addressed object stores,"** never a bespoke sync protocol bolted onto the object model from outside. A local repository additionally keeps **remote-tracking refs** (`refs/remotes/origin/main`, conventionally) — a local, read-only record of where the *last known* state of the remote's `main` was, updated on every fetch, which is what lets `git status` report "3 commits behind origin/main" without a network round trip on every single status check.

---

# 45. The Push/Pull Negotiation: Which Objects Does the Remote Not Have?

```java
public final class PushNegotiator {

    private final ObjectStore localObjectStore;

    public PushNegotiator(ObjectStore localObjectStore) { this.localObjectStore = localObjectStore; }

    /** Returns exactly the objects the remote is missing -- computed LOCALLY, before a single byte is sent. */
    public Set<String> objectsToSend(String localRefTip, Set<String> remoteHasHashes) throws IOException {
        Set<String> reachableFromLocal = reachableFrom(localRefTip); // §35's ancestor-walk, reused verbatim
        reachableFromLocal.removeAll(remoteHasHashes); // set difference -- everything the remote ALREADY has drops out
        return reachableFromLocal;
    }

    private Set<String> reachableFrom(String commitHash) throws IOException {
        Set<String> reachable = new HashSet<>();
        Deque<String> stack = new ArrayDeque<>(List.of(commitHash));
        while (!stack.isEmpty()) {
            String current = stack.pop();
            if (!reachable.add(current)) continue;
            Commit commit = (Commit) GitObjectParser.parse(localObjectStore.read(current));
            reachable.add(commit.treeHash()); // the commit's tree, and (recursively, elided) every blob/subtree it reaches
            for (String parent : commit.parentHashes()) stack.push(parent);
        }
        return reachable;
    }
}
```

The negotiation, at a high level: the client tells the server which refs it has and their tips; the server (or client, for a pull) computes exactly which commits are reachable from the *wanted* tip but **not** reachable from anything the *other* side already has — precisely a set-difference over two reachability sets, both computed with the identical ancestor-walk `MergeBaseFinder` (§35) already built. Only that computed difference is ever transferred, which is why pushing 3 new commits to a repository with 500,000 existing ones transfers roughly 3 commits' worth of new objects, never the other 500,000.

---

# 46. Implementing a Simple Push

```java
public final class SimplePush {

    private final ObjectStore localObjectStore;
    private final PushNegotiator negotiator;
    private final RemoteTransport transport; // sends objects + updates a remote ref, over the network

    public SimplePush(ObjectStore localObjectStore, PushNegotiator negotiator, RemoteTransport transport) {
        this.localObjectStore = localObjectStore;
        this.negotiator = negotiator;
        this.transport = transport;
    }

    public void push(String refName, String localTip) throws IOException {
        Set<String> remoteHasHashes = transport.queryRemoteObjectHashes(); // what the remote already reports having
        Set<String> toSend = negotiator.objectsToSend(localTip, remoteHasHashes); // §45

        for (String hash : toSend) {
            transport.sendObject(hash, localObjectStore.read(hash)); // §16's stored bytes, transferred as-is
        }
        transport.updateRemoteRef(refName, localTip); // the LAST step -- only after every object landed safely
    }
}
```

Updating the remote's ref only **after** every needed object has been transferred is the push equivalent of the [TinyDB Storage Engine guide's WAL-before-flush rule](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>): never let a pointer become visible to anyone pointing at data that isn't durably, completely present yet — a push that transfers objects but crashes before updating the ref leaves the remote's history exactly as it was before (safe, if incomplete); a push that updated the ref *first* and then failed mid-transfer would leave a ref pointing at commits the remote doesn't actually have the objects for, a corrupted, unrecoverable state.

---

# 47. Follow-up Question 13 — "You've Been Committing for Years. The Repository Is Huge. What Breaks First, Storage-Wise?"

> **Interviewer:** *"Every commit stores a full Tree, and every changed file is a full new Blob. A large binary asset changed 500 times over the project's life is now stored 500 times, nearly in full each time. What's the fix?"*

§48 names the cost precisely; §49-§50 fix it the way real Git does — not by storing less content, but by storing the **differences** between similar objects instead of full copies.

---

# 48. Why Storing Every Blob in Full, Forever, Doesn't Scale

§15-§16's `ObjectStore` deduplicates **identical** content perfectly — two commits with the byte-for-byte same file share one Blob. It does nothing at all for **similar-but-not-identical** content: a 10MB file with one line changed produces a brand-new, entirely separate 10MB Blob, sharing not a single stored byte with the previous version, even though the two are 99.9% identical. Across a long-lived repository's full history, this is where the overwhelming majority of storage actually goes — not distinct content, but many *near-duplicate* versions of the same evolving files.

---

# 49. Delta Compression and Packfiles

A **packfile** stores a set of related objects together, where most of them are stored not as full content but as a **delta** — a compact description of how to reconstruct this object's bytes given some *other*, already-stored object (a "base") plus a small set of copy/insert instructions. Git's own heuristic for choosing which objects to delta against which base is itself a substantial piece of engineering (grouping by similar size and path name is a good starting heuristic) — this guide names that selection problem honestly rather than pretending it's trivial, and implements the simpler, foundational piece beneath it: given a chosen base and target, compute the actual delta.

---

# 50. Implementing a Simple Delta Encoder

```java
public record DeltaInstruction(boolean isCopy, int sourceOffset, int length, byte[] insertedBytes) { }

public final class DeltaEncoder {

    private static final int MIN_MATCH_LENGTH = 16; // shorter matches cost more to encode than they save

    /** A simplified delta: find the longest runs of BASE bytes that reappear in TARGET, encode those as
     *  copy instructions, and encode everything else as literal insert instructions. */
    public List<DeltaInstruction> encode(byte[] base, byte[] target) {
        Map<ByteSequence, Integer> baseIndex = indexFixedSizeChunks(base, MIN_MATCH_LENGTH);
        List<DeltaInstruction> instructions = new ArrayList<>();
        int targetPos = 0;

        while (targetPos < target.length) {
            Optional<Integer> matchInBase = findLongestMatch(baseIndex, base, target, targetPos);
            if (matchInBase.isPresent()) {
                int matchLength = extendMatch(base, target, matchInBase.get(), targetPos);
                instructions.add(new DeltaInstruction(true, matchInBase.get(), matchLength, null));
                targetPos += matchLength;
            } else {
                int literalEnd = Math.min(targetPos + MIN_MATCH_LENGTH, target.length);
                instructions.add(new DeltaInstruction(false, -1, 0, Arrays.copyOfRange(target, targetPos, literalEnd)));
                targetPos = literalEnd;
            }
        }
        return instructions;
    }

    public byte[] decode(byte[] base, List<DeltaInstruction> instructions) {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        for (DeltaInstruction instruction : instructions) {
            if (instruction.isCopy()) {
                out.write(base, instruction.sourceOffset(), instruction.length());
            } else {
                out.writeBytes(instruction.insertedBytes());
            }
        }
        return out.toByteArray();
    }

    // indexFixedSizeChunks(...)/findLongestMatch(...)/extendMatch(...) -- a rolling-hash chunk index over
    // `base`, letting the encoder find a candidate match position for any window of `target` in roughly
    // constant time per position, rather than an O(n*m) brute-force scan.
}
```

For the "one line changed in a 10MB file" case from §48, this produces roughly: one long copy instruction for everything before the changed line, one small insert instruction for the new line's bytes, and one long copy instruction for everything after — a delta of a few dozen bytes describing a 10MB object, rather than storing another 10MB in full. `decode` is deliberately trivial by comparison — nearly all of the engineering complexity of delta compression lives entirely in `encode`, finding good matches efficiently, never in reconstructing from them.

---

# 51. Follow-up Question 14 — "Deleted Branches Leave Orphaned Commits. How Do You Reclaim That Space Safely?"

> **Interviewer:** *"A branch gets deleted, or a rebase (§41-§42) creates new commits and abandons the old ones. Those old objects are still sitting in the ObjectStore. How do you safely reclaim that space, without ever deleting something still needed?"*

§52-§53 answer with **reachability from refs** — the exact same idea `PushNegotiator` (§45) already computes for a completely different reason, reused here to answer "what's safe to delete" instead of "what needs to be sent."

---

# 52. Garbage Collection: Reachability From Refs

An object is **reachable** if it can be reached by starting from some ref (every branch tip, every tag, and — conservatively — anything reflog-recorded as a recent `HEAD` position) and following commit-parent, tree-entry, and tag-target pointers outward. Anything **not** reachable from any of those starting points can never be referenced by any future operation — no branch points at it, no tag points at it, nothing in the DAG that anything reachable depends on includes it — and is therefore safe to delete. This is a textbook **mark-and-sweep** algorithm: mark every object reachable from a root set, then sweep (delete) everything that was never marked.

---

# 53. Implementing Mark-and-Sweep Garbage Collection

```java
public final class GarbageCollector {

    private final ObjectStore objectStore;
    private final RefStore refStore;

    public GarbageCollector(ObjectStore objectStore, RefStore refStore) { this.objectStore = objectStore; this.refStore = refStore; }

    public GcReport collect() throws IOException {
        Set<String> reachable = markReachableFromAllRefs(); // §52's "mark" phase
        Set<String> allStoredHashes = objectStore.allObjectHashes();

        Set<String> unreachable = new HashSet<>(allStoredHashes);
        unreachable.removeAll(reachable); // §52's "sweep" phase -- everything left over

        for (String hash : unreachable) {
            objectStore.delete(hash); // the ONLY place in this entire guide an object is ever physically removed
        }
        return new GcReport(reachable.size(), unreachable.size());
    }

    private Set<String> markReachableFromAllRefs() throws IOException {
        Set<String> reachable = new HashSet<>();
        for (String refTip : refStore.allRefTips()) { // every branch AND every tag -- the full root set
            reachable.addAll(reachableFrom(refTip, objectStore)); // §35/§45's identical ancestor-and-tree walk, reused a third time
        }
        return reachable;
    }
}
```

This is now the **third** distinct place this guide reuses the same underlying "walk everything reachable from a starting hash" traversal — `MergeBaseFinder` (§35) for finding a merge base, `PushNegotiator` (§45) for negotiating what to send, and `GarbageCollector` (§53) for deciding what's safe to delete. All three are the identical graph-reachability operation, applied to answer three completely different questions — which is exactly the kind of reuse that falls out naturally once the underlying data structure (a hash-linked DAG) is modeled once, correctly, rather than each feature growing its own bespoke traversal.

---

# 54. Follow-up Question 15 — "How Do You Make `git log` and `git status` Fast on a Repository With a Million Commits?"

> **Interviewer:** *"Every algorithm you've shown me walks the commit graph by reading one commit object, parsing its parents, reading the next one, and so on. On a repository with a million commits, that's a million small object-store reads for one `git log`. How do you make that fast?"*

§55 answers with exactly the kind of pre-computed, denormalized structure this guide's other companion guides reach for repeatedly when a per-item computation gets too expensive to redo from scratch on every call.

---

# 55. Performance at Scale: Commit-Graph Caching and Bitmap Indexes

A **commit-graph file** pre-computes and caches, for every commit, the information §35's `MergeBaseFinder` and §52's `GarbageCollector` otherwise have to re-derive by repeatedly reading and parsing raw commit objects: each commit's parents (as direct array indices into the cache, not hashes needing a further lookup), and a generation number (roughly, "how many commits deep is this from the root") that lets an LCA search prune huge portions of history it can prove are too shallow or too deep to matter, without visiting them at all. A **reachability bitmap index** goes further for GC (§52-§53) and push negotiation (§45) specifically: a precomputed bitmap, one bit per object, marking exactly what's reachable from a small set of frequently-queried refs (typically every branch tip), refreshed periodically rather than walked fresh on every single GC run or push. Both are the same underlying idea applied twice: **trade a periodically-refreshed, precomputed structure for the cost of re-deriving the same graph facts from raw objects on every single query** — the identical tradeoff [the API Gateway guide's hybrid rate limiter](<Design an API Gateway and a Rate Limiter — HLD, LLD, and Class Design From Scratch.md>) makes between a per-request network round trip and a periodically-refreshed local cache.

---

# 56. Full Worked Example: A Complete Commit-Branch-Merge Cycle, End to End

```text
1.  git add README.md                     -> Index.stage("README.md", blobHash, "100644")             (§29)
2.  git commit -m "Initial commit"        -> Index.buildTreeFromStagedEntries() -> treeHash            (§29)
                                              CommitBuilder.commit(treeHash, [], "Arpan", "Initial commit") -> c1  (§22)
                                              RefStore.setRef("refs/heads/main", c1)                    (§25)
3.  git checkout -b feature-x             -> RefStore.setRef("refs/heads/feature-x", c1)                (§25)
                                              Head.pointToRef("refs/heads/feature-x")                    (§25)
4.  (edit a file, on feature-x)
    git add . && git commit -m "Add X"    -> new Tree, new Commit f1 (parent: [c1])                     (§22)
                                              RefStore.setRef("refs/heads/feature-x", f1)
5.  (meanwhile, on main: a separate commit c2, parent [c1])
6.  git checkout main && git merge feature-x
       MergeBaseFinder.findMergeBase(c2, f1) -> c1                                                       (§35)
       ThreeWayMerge.merge(treeOf(c1), treeOf(c2), treeOf(f1)) -> clean merge, no conflict (§36's case 4/5)  (§37)
       new Commit m (parents: [c2, f1])                                                                  (§22)
       RefStore.setRef("refs/heads/main", m)                                                             (§25)
7.  git push origin main
       PushNegotiator.objectsToSend(m, remoteHasHashes) -> {m, f1, and their new trees/blobs only}        (§45)
       SimplePush.push("refs/heads/main", m) -> objects transferred, THEN remote ref updated               (§46)
```

Every numbered line traces to a section this guide built real code for — including step 6's merge, which required §35's LCA (finding `c1`) before §37's three-way logic could run at all.

---

# 57. Final Architecture Diagram

```text
              Working Directory  --git add-->  Index (§28-§29)  --git commit-->  ObjectStore (§16)
                                                                                         |
                                                                        content-addressed: Blob/Tree/Commit
                                                                                         |
                                            RefStore + HEAD (§24-§25)  <---- points into the DAG above
                                                         |
                     +-----------------------------------+-----------------------------------+
                     v                                    v                                    v
          MergeBaseFinder (§35)                ThreeWayMerge (§37)                 Rebaser (§42, reuses §37)
          LCA via ancestor-set +                five-case decision                  replays commits onto
          reverse BFS                           structure, per file                  a new base
                     |
                     v
          PushNegotiator (§45) / GarbageCollector (§52-§53)
          BOTH reuse the identical reachability-walk pattern §35 already built
                     |
                     v
          Packfile + DeltaEncoder (§49-§50)         Commit-Graph Cache / Bitmap Index (§55)
          storage efficiency for similar objects     precomputed graph facts for speed at scale
```

---

# 58. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Composite** | `Tree`/`Blob` (§18-§19) | A directory snapshot uniformly contains files and subdirectories, hashed recursively the same way at every level. |
| **Strategy** | `DiffAlgorithm` (§32), merge/rebase reusing the same `ThreeWayMerge` (§42) | The diff algorithm and the merge-conflict logic are both swappable behind one interface each. |
| **Memento (conceptually)** | `Commit` (§21-§22) | Each commit is an immutable, complete snapshot reference a repository can always restore to, never a mutable, in-place-edited record. |
| **Content-Addressable Storage** | `ObjectStore` (§15-§16) | Identity IS the hash of content — deduplication and tamper-evidence both fall out of this one property for free. |
| **Command (conceptually)** | Each Git operation (commit, merge, rebase) | Every operation is a discrete, well-defined transformation from one repository state to the next, built from the same small set of primitives (§16, §25). |
| **Mark-and-Sweep** | `GarbageCollector` (§52-§53) | The standard reachability-based reclamation algorithm, applied to a hash-addressed object graph instead of a language runtime's heap. |

---

# 59. SOLID Principles Applied

- **Single Responsibility**: `ObjectStore` only stores/retrieves bytes by hash; `RefStore` only manages named pointers; `MergeBaseFinder` only computes an LCA. None of them know how to serialize a `Tree` or format a commit message.
- **Open/Closed**: adding a new `DiffAlgorithm` implementation (§32), or a new object type beyond Blob/Tree/Commit/Tag (a signed commit, say), extends the `sealed interface GitObject` (§16) and its exhaustive `switch` sites deliberately — the compiler forces every dispatch site to be updated, but no *existing* class needs to change its own logic to accommodate the new case.
- **Liskov Substitution**: any `GitObject` — `Blob`, `Tree`, `Commit`, `Tag` — is fully substitutable wherever `ObjectStore.store(GitObject)` (§16) is called; the store never needs to know which concrete type it's persisting.
- **Interface Segregation**: `DiffAlgorithm` exposes exactly one method (`diff`); `ObjectStore` exposes exactly `store`/`read` — neither forces an implementation to support behavior it doesn't need.
- **Dependency Inversion**: `ThreeWayMerge` (§37) depends on the `DiffAlgorithm` interface, injected through its constructor, never on `MyersDiff` directly — swapping in a different diff algorithm requires zero changes to the merge logic.

---

# 60. Common Mistakes When Building This Yourself

- **Hashing a Tree's entries in insertion order instead of sorted order** (§19) — two functionally identical directory snapshots silently hash differently depending on iteration order, quietly breaking deduplication in a way that's easy to miss until storage usage looks mysteriously high.
- **Comparing only two versions during a merge, without a common-ancestor base** (§34-§36) — makes it structurally impossible to distinguish "both branches coincidentally made the identical edit" from "one branch added something the other never touched," producing false conflicts or, worse, silently dropped changes.
- **Walking only the first-parent chain when looking for a merge base** (§35) — silently wrong the moment either branch's history includes an earlier merge commit, which is not a rare edge case in any real, actively-developed repository.
- **Updating a remote's ref before confirming every needed object actually transferred** (§46) — the exact ordering mistake the [TinyDB Storage Engine guide's WAL-before-flush rule](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) warns about, here capable of leaving a remote repository in a state where a ref points at commits whose objects were never actually received.
- **Deleting an object during garbage collection based on an incomplete root set** (§52) — forgetting to include tags, or reflog entries, or any ref type in the reachability walk risks deleting an object something still legitimately needs.
- **Treating rebase as "just moving commits"** (§40-§41) — every replayed commit gets a genuinely new hash and can genuinely conflict; a rebase that's already been pushed and pulled by someone else rewrites history out from under them, a real, well-known operational hazard worth stating explicitly, not glossed over.

---

# 61. Testing Strategy

- **`ObjectStore`** (§16): storing identical content twice produces the identical hash and writes the underlying file only once; storing different content produces different hashes; reading back any stored object's bytes matches exactly what was stored.
- **`Tree`** (§19): two `Tree` objects built from the same entries added in a *different* order produce identical serialized bytes and therefore identical hashes.
- **`MyersDiff`** (§32): a known before/after pair of line lists produces the exact expected sequence of `Unchanged`/`Added`/`Removed` lines; an identical pair of lists produces zero `Added`/`Removed` entries.
- **`MergeBaseFinder`** (§35): a DAG containing an earlier merge commit still returns the correct, lowest common ancestor — this is the test that would have caught the first-parent-only mistake named in §60.
- **`ThreeWayMerge`** (§37): each of §36's five cases is exercised directly — only-A-changed, only-B-changed, both-changed-identically (clean), and both-changed-differently (a genuine, correctly-flagged conflict).
- **`GarbageCollector`** (§52-§53): an object reachable only through a *tag* (not a branch) survives collection; a genuinely unreachable object (from a deleted branch, with no other ref pointing at its history) is correctly removed.
- **`DeltaEncoder`** (§50): `decode(base, encode(base, target))` reconstructs `target` exactly, for both a near-identical pair of inputs and a pair sharing no common content at all (falling back to entirely literal inserts).

---

# 62. Suggested Future Enhancements

- **Line-range/hunk-level conflict detection** (§37) — narrowing a merge conflict down to the specific overlapping lines within a file, rather than this guide's simplified whole-file granularity.
- **Diff heuristics for very large files** (§32) — a sliding window and long-identical-run hashing, the standard refinement over the pure `O(m*n)` LCS table for files where that complexity genuinely matters.
- **Binary file diffing** (§6's stated scope limitation) — this guide's `MyersDiff` (§32) operates on lines of text; a binary-aware diff needs a fundamentally different, byte-oriented comparison (or, more commonly, simply presenting a binary file as "changed" without a line-level diff at all, which is what most real tools do by default).
- **A real base-selection heuristic for delta compression** (§49) — grouping candidate objects by similar size and path before attempting delta encoding, rather than this guide's simplified "given a chosen base" starting point.
- **Signed commits and tags** — extending the `sealed interface GitObject` (§16, §59's Open/Closed example) with a cryptographic signature over a commit's own hash, proving authorship beyond what content-addressing alone establishes.
- **Partial/shallow clones** — deliberately relaxing §13's "every clone is a complete copy" guarantee for genuinely enormous repositories, trading some offline capability for a dramatically smaller initial clone.
- **A real network transport protocol** for §44-§46's push/pull negotiation, with proper authentication, rather than this guide's `RemoteTransport` abstraction left intentionally unspecified.

---

# 63. Progressive Interview Question Set

1. Why does making an object's storage key the hash of its own content automatically deduplicate identical content, with no separate index required?
2. Why is SHA-256 the right choice for object identity here, when a much cheaper CRC32 is sufficient for a database page's corruption check?
3. Walk through why `Tree.serialize()` must sort entries deterministically, and what silently breaks if it doesn't.
4. Why can the commit DAG never contain a cycle, structurally — not "we check for one," but why one cannot be constructed in the first place?
5. What, precisely, does creating a branch actually do at the storage level, and why does that make it instant regardless of repository size?
6. Why does a correct merge-base algorithm need to consider the FULL ancestor set, not just each branch's first-parent chain?
7. Explain the five-case decision structure of a three-way merge, and why a plain two-way diff between the two branch tips can't distinguish a coincidental identical edit from a real conflict.
8. Why is rebasing a `git merge` in disguise, structurally — what specific piece of the merge machinery does a rebase reuse, and how?
9. Name the three distinct places in this guide's design that all reduce to the same graph-reachability computation, and explain why that reuse falls out naturally rather than being a coincidence.
10. If asked to add support for signed commits, walk through exactly what would need to change in `GitObject`'s sealed hierarchy, and what would NOT need to change anywhere else.

---

# 64. Final Takeaway

Every genuinely hard problem in a version control system reduces to one of two things: a question about a **hash-addressed, immutable graph** (what's this object's identity, what's reachable from here, where did these two histories diverge), or a question about **comparing content** (what changed, and when two changes collide, which one wins). Once an object's identity is honestly just the hash of its own bytes, deduplication, tamper-evidence, cheap branching, and a diff-free way to detect "nothing changed here" all fall out as consequences of that one decision, not as separate features each requiring their own mechanism. The genuinely hard engineering — a correct LCA on a real DAG, a correct three-way merge, a delta encoder that actually saves space — is real, algorithmic work, and this guide's central claim is that none of it is optional or hand-wavable if the goal is a version control system that's actually correct, not merely one that looks correct on a whiteboard.

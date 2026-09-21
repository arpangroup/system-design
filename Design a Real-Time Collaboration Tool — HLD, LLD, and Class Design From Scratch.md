# Design a Real-Time Collaboration Tool — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a real-time collaborative document editor — like Google Docs or Figma's multiplayer editing. Cover both high-level design (architecture, real-time transport, scaling) and low-level design (the actual algorithm that resolves concurrent edits, and its class model). Be ready to justify every design decision when I push back."*
>
> This guide is structured exactly as that interview unfolds: requirements gathering, a high-level architecture built up decision by decision, a deep low-level dive into **both** major approaches to concurrent editing (Operational Transformation and CRDTs), and a sequence of escalating **follow-up questions** — each answered with real reasoning and working code, not a diagram presented as if no one ever questioned it.

---

# 1. What We Are Building

We are building **MiniCollab** — a real-time collaborative document editor covering:

- **Functional requirements**: multiple users editing the same document simultaneously, live cursors and presence, sharing with view/comment/edit permissions, undo/redo, and version history.
- **High-level design**: a client-server real-time architecture, the transport layer (WebSockets), session affinity for scaling, offline support and reconciliation, and horizontal scaling of the real-time layer.
- **Low-level design**: two complete, contrasting engines for resolving concurrent edits — **Operational Transformation** (the algorithm behind Google Docs) and **CRDTs** (the algorithm behind Figma and many peer-to-peer-capable tools) — built as real, working class models, not just named.
- **Consistency and failure**: exactly what guarantee the system provides when two edits collide, what happens when the server crashes mid-broadcast, and how the design scales to millions of concurrently-edited documents.

```text
                     User A's browser              User B's browser
                     (local edit applied            (local edit applied
                      IMMEDIATELY, optimistic)        IMMEDIATELY, optimistic)
                            |                                |
                            |     WebSocket (§15)            |
                            v                                v
                    +----------------------------------------------+
                    |         Collaboration Server (owns this        |
                    |         document's session, §23)               |
                    |  Operational Transform OR CRDT engine (§25-§37)|
                    +----------------------------------------------+
                                        |
                                        v
                          Document store (source of truth, §38)
                          + Operation log (durability, §53)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain, precisely, why concurrent editing of shared text is a fundamentally harder problem than concurrent editing of a database row, and why last-write-wins is the wrong answer.
- Implement a working **Operational Transformation** engine: representing edits as operations, and transforming one operation against another so both replicas converge on the same final document.
- Implement a working **character-level CRDT**: unique, order-preserving identifiers that let replicas converge without any central coordination at all.
- Justify choosing OT over CRDTs (or vice versa) for a given product's requirements, from real tradeoffs, not a memorized preference.
- Design live presence (cursors, selections, "who's online") as a genuinely separate concern from document content synchronization.
- Reason about what breaks first at scale (millions of documents, hundreds of thousands of concurrent editors) and name the specific technique that addresses each bottleneck.

---

# 3. Why This Matters (The Interview, Framed)

Designing a real-time collaborative editor is a favorite **senior/staff systems interview question** specifically because "just use a database and lock the row" — the correct answer to many other system-design prompts — is actively wrong here, and recognizing *why* is the actual signal being tested:

- **The core problem is algorithmic, not just architectural.** Unlike [the JIRA-style guide's optimistic locking](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>), which resolves a conflict by rejecting the second writer, a collaborative editor **must** merge both users' concurrent keystrokes into one coherent document — rejecting either user's input is not an acceptable product experience.
- **Two, genuinely different, well-established solutions exist** (Operational Transformation and CRDTs), each with real tradeoffs — a strong answer can build and compare both, not just name-drop one.
- **Real-time transport and presence are their own design problems**, distinct from the conflict-resolution algorithm itself, and conflating them is a common, revealing mistake.
- **Consistency guarantees must be stated precisely** — "eventually consistent, guaranteed convergence" is a specific, defensible claim; "it just syncs" is not.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and algorithm implementations. |
| Real-time transport | WebSockets | A persistent, bidirectional connection — essential for low-latency operation broadcast (§15). |
| Document storage | A relational or document store for the durable document state | The document's current content and metadata need durable, queryable storage (§38). |
| Operation log | A durable, ordered append log (Kafka-style, or [TinyDB's own WAL](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>)) | Every applied operation is logged before being acknowledged, exactly the durability discipline that guide already establishes (§53). |
| Pub/Sub backbone | A message broker supporting topic-based fan-out | Broadcasts operations to every connected client editing the same document (§56). |

---

# 5. Project Structure

```text
minicollab/
├── src/main/java/com/example/minicollab/
│   ├── ot/
│   │   ├── Operation.java, InsertOp.java, DeleteOp.java // §26
│   │   ├── OperationTransformer.java     // §27-§29
│   │   └── OtDocumentSession.java        // §30
│   ├── crdt/
│   │   ├── CrdtCharacter.java, CharacterId.java // §34-§35
│   │   └── CrdtDocument.java              // §35
│   ├── document/
│   │   ├── Document.java, DocumentSnapshot.java // §38, §49
│   │   └── VersionHistory.java            // §49
│   ├── session/
│   │   ├── EditSession.java               // §39
│   │   ├── PresenceTracker.java           // §41
│   │   └── CursorPosition.java
│   ├── sharing/
│   │   ├── ShareRole.java, DocumentAccess.java // §44
│   │   └── AccessChecker.java
│   └── undo/
│       └── UndoManager.java              // §47
└── src/test/java/com/example/minicollab/
    ├── OperationalTransformTest.java
    ├── CrdtConvergenceTest.java
    └── ConcurrentEditStressTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"What's actually being edited — plain text, rich text with formatting, or structured content like a spreadsheet or a design canvas? Do we need to support offline editing that later reconciles? Is this purely client-server, or does it need to work peer-to-peer? How many concurrent editors on one document is realistic — a handful, or thousands watching a live event doc?"*

Exactly as [the JIRA guide's opening](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) and [the file storage guide's opening](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) both establish, narrowing the prompt before designing is the correct first move — the answers here directly determine whether Operational Transformation or CRDTs (§17, §32) is the better starting point, and whether offline support (§19-§20) is core or out of scope.

For this guide, we settle on a concrete scope: **plain/rich text documents**, **client-server** (not peer-to-peer), supporting **offline edits with reconciliation on reconnect**, targeting **dozens of concurrent editors per document** (Google Docs' actual realistic range, not thousands).

---

# 7. Functional Requirements

- Multiple users edit the **same document** simultaneously, with each user's edits appearing live for everyone else within a short, bounded delay.
- Every connected user sees **live cursors and selections** from other active editors.
- Users can work **offline** and have their edits correctly merged in when they reconnect.
- Documents can be **shared** with specific permission levels: Viewer, Commenter, Editor.
- **Undo/redo** works correctly even when other users have made changes in between.
- A document's **version history** is browsable and restorable.
- Comments can be attached to specific ranges of content.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Low latency** | A keystroke should appear on other users' screens within roughly 100ms under normal conditions | Optimistic local application (§13) + a lightweight WebSocket transport (§15) |
| **Convergence** | Every replica (every connected client, and the server) must eventually reach the **exact same** document content, regardless of the order operations arrive in | The actual mathematical guarantee OT (§27) and CRDTs (§33) are each built to provide |
| **Availability during network issues** | A user shouldn't lose work or be blocked from typing just because their connection is briefly unstable | Offline editing and reconciliation (§19-§20) |
| **Durability** | A confirmed edit must survive a server crash | An operation log, exactly [TinyDB's WAL discipline](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) (§53) |
| **Scalability** | Millions of documents, each with its own small set of concurrent editors | Session affinity (§23) and document sharding (§55) — scaling by *number of documents*, not by making one document's editor count unbounded |

---

# 9. Follow-up Question 1 — "What's the Core Challenge Here, Fundamentally?"

> **Interviewer:** *"Forget architecture for a second. In one sentence: what makes this problem fundamentally different from, say, two people editing the same database row?"*

The honest answer: with a database row, **rejecting** the second concurrent writer (optimistic locking, [as the locking guide builds in full](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>)) is an acceptable outcome — the user retries. With collaborative text editing, **rejecting a keystroke is not an acceptable product experience** — if Alice and Bob type at the same time, both of their keystrokes must end up in the final document, correctly interleaved, with **neither** user ever seeing their own typing silently vanish or getting an error mid-sentence. The entire discipline of Operational Transformation and CRDTs exists to solve exactly this: **merge**, don't reject.

---

# 10. The Core Challenge: Concurrent Edits to Shared State

Concretely: Alice and Bob both start with the document `"cat"`. Alice inserts `"h"` at position 0, producing `"hcat"` on her screen instantly (optimistic local application, §13). Bob, at the same instant, inserts `"s"` at position 3, producing `"cats"` on his screen instantly. Both edits must reach the other user and the server, and **all three copies** (Alice's, Bob's, and the server's) must converge on the same final string — `"hcats"` — even though Bob's insert position (3) was computed against the original `"cat"`, not against Alice's already-modified `"hcat"`. Naively applying Bob's `insert(3, "s")` against `"hcat"` would produce `"hcast"` — **wrong**, because position 3 in `"hcat"` is a different character than position 3 was in `"cat"`. This exact positional-drift problem is what §17 onward exists to solve.

---

# 11. High-Level Architecture Overview

At the highest level: each client applies its own edits **immediately and optimistically** (never waiting for a server round trip to show the user their own keystroke), sends the edit to a **Collaboration Server** that owns that document's live session, which resolves it against any concurrently-arriving edits from other clients and broadcasts the resolved result to everyone connected — including, if needed, a correction back to the original author. §1's diagram is this section's answer, restated for reference; §12–§23 justify every piece of it.

---

# 12. Follow-up Question 2 — "Client-Server or Peer-to-Peer?"

> **Interviewer:** *"Real-time collaboration sounds like it could be peer-to-peer — why route everything through a server at all?"*

Peer-to-peer is architecturally possible (and is exactly what CRDTs, §32-§37, are specifically designed to support without any central coordinator) — but a central server buys three things a pure peer-to-peer mesh doesn't get for free: a **single source of truth** for the document's durable state (§38) even when every client disconnects, a **simple place to enforce permissions** (§43-§44) before an edit is even accepted, and — critically for Operational Transformation specifically (§17) — a **canonical ordering** of operations, which OT's classic algorithm actually depends on (§30). This guide picks client-server as the default architecture (matching §6's scoping), while §32-§37 show that CRDTs, as an alternative *algorithm* choice within that same client-server architecture, could later support peer-to-peer without redesigning the whole system.

---

# 13. Why a Central Server (Even Though It's "Real-Time")

The word "real-time" doesn't mean "server-free" — it means **low-latency**, which the design achieves through **optimistic local application**: a keystroke updates the local editor's screen instantly, in parallel with (not waiting for) sending it to the server. The server's round trip only affects *other* users' screens and, in the rare event of a real conflict, a correction to the *originating* user's own document — the typing experience itself never waits on the network, which is what actually makes the system feel real-time despite a server sitting in the middle of every edit.

---

# 14. Follow-up Question 3 — "How Does a Client Know What Changed?"

> **Interviewer:** *"Bob is looking at the document. Alice types a sentence on the other side of the world. How does that text actually appear on Bob's screen?"*

Not by Bob's client polling the server on a timer (the same latency-vs-bandwidth tradeoff [the file storage guide's real-time discussion](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already rejects for board updates) — a **persistent WebSocket connection** per active editor, with the server pushing each resolved operation to every other connected client the instant it's ready.

---

# 15. WebSockets and the Real-Time Transport Layer

```text
Client A                    Collaboration Server                    Client B
   |  1. WebSocket connect, join document "doc-42"'s session  ------------>|
   |                                                                        |
   |  2. Alice types "h" -> InsertOp(pos=0, char='h') sent over WebSocket   |
   |------------------------------------------------------------------->  |
   |                                    3. Server resolves/transforms (§27)|
   |                                    4. Broadcasts resolved op to ALL   |
   |                                       OTHER connected clients          |
   |<-------------------------------------------------------------------  |
   |                                                        (Bob's screen updates)
```

Each document's session is a lightweight, in-memory broadcast group — every client connected to that document receives every other client's resolved operations over its own persistent connection, with no polling and no unnecessary round trips for clients not editing that specific document.

---

# 16. Follow-up Question 4 — "Two Users Type at the Same Position at the Same Time. What Happens?"

> **Interviewer:** *"Walk me through, mechanically, what the server actually does when it receives two operations that were both computed against the same starting document state."*

This is §10's exact scenario, now asked as a direct algorithmic question rather than a motivating example — and it's the pivot point into the two real solutions this guide builds in full: **transform** one operation against the other so it's still correct when applied to the *already-modified* document (Operational Transformation, §17), or design the data structure so that **applying operations in any order produces the same result automatically** (CRDTs, §33).

---

# 17. Two Approaches: Operational Transformation vs CRDTs

| | Operational Transformation (§25-§31) | CRDTs (§32-§37) |
|---|---|---|
| Core idea | Transform a concurrent operation's position/content so it's correct when applied after another operation, rather than the state it was originally computed against | Design the data itself (e.g. unique, order-preserving IDs per character) so that applying operations in **any** order converges to the same result — no transformation step needed |
| Needs a central sequencer? | Yes, in the classic algorithm — a server (or an agreed-upon order) is what makes transformation well-defined (§30) | No — this is CRDTs' headline property, enabling true peer-to-peer (§32) |
| Complexity of the core algorithm | The transform functions (§28-§29) have many cases and are notoriously easy to get subtly wrong | The data structure itself is simple; the complexity moves into memory overhead (tombstones, §35) instead of transform logic |
| Real-world example | Google Docs (its classic architecture) | Figma, Yjs-based tools, many offline-first apps |

Both are correct, production-proven answers — building both, and being able to state exactly what each trades off, is a far stronger interview answer than picking one and treating the other as inferior.

---

# 18. Document Storage and Persistence

The document's **current, durable content** lives in a standard data store (§38) — separate from the live, in-memory OT/CRDT session state that only exists while at least one client is actively connected. When the last client disconnects, the session's final resolved state is what gets persisted — the next time anyone opens the document, a fresh session is initialized **from that durable state**, not from an empty document, exactly the way [TinyDB's crash recovery](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) reconstructs state from durable storage rather than assuming it's always live in memory.

---

# 19. Follow-up Question 5 — "How Do You Handle a User Going Offline and Coming Back?"

> **Interviewer:** *"Alice's laptop loses wifi for two minutes. She keeps typing the whole time — her editor doesn't know she's offline. What happens when her connection comes back?"*

Her client must have kept a **local queue of unacknowledged operations** generated while disconnected — on reconnect, it replays that queue to the server, which resolves each one (via OT transformation against everything that happened while she was gone, or, for CRDTs, simply merges them in, since CRDT operations are commutative by design) exactly as if they'd arrived one at a time in real time, just delayed.

---

# 20. Offline Support and Reconciliation

```java
public class OfflineOperationQueue {
    private final Deque<Operation> pendingOperations = new ArrayDeque<>();

    public void recordLocalOperation(Operation op) {
        pendingOperations.addLast(op); // applied locally immediately (§13), queued for later transmission
    }

    public void onReconnect(WebSocketSession session) {
        while (!pendingOperations.isEmpty()) {
            session.send(pendingOperations.pollFirst()); // replayed in the ORDER they were originally made — preserves Alice's intent
        }
    }
}
```

Preserving the **order** Alice's own operations were originally made in is essential — replaying them out of order (or, worse, coalescing them incorrectly) could reconstruct a different edit than what she actually typed, even if the final character-by-character content happened to look similar.

---

# 21. Presence: Cursors, Selections, and Who's Online

Presence — whose cursor is where, who has a range selected, who's currently viewing the document at all — is **explicitly not** part of the document's actual content, and must not flow through the same OT/CRDT conflict-resolution pipeline (§17) that content edits do: two cursors can occupy overlapping visual space with zero conflict to resolve, unlike two concurrent text insertions. Presence is a lighter-weight, ephemeral broadcast — "here's my current cursor position" — sent frequently, never persisted, and simply overwritten by each user's next update, architecturally simpler than content synchronization precisely because it doesn't need convergence guarantees at all.

---

# 22. Follow-up Question 6 — "How Do You Scale to Millions of Documents Being Edited Concurrently?"

> **Interviewer:** *"You've designed this well for one document. Now there are 5 million documents, each with a handful of active editors. How does that change your architecture?"*

The key realization: this workload scales by **document count**, not by concurrent-editor count *per* document (§6's scoping already assumes a realistic, bounded number of simultaneous editors on any single document) — which means the right scaling lever is spreading **documents**, not individual editors, across many servers.

---

# 23. Session Affinity: Routing a Document's Edits to One Owning Server

```text
Client connects, requests document "doc-42"
        |
        v
Routing layer: hash("doc-42") -> Collaboration Server #7   (consistent hashing, same technique as
        |                                                    the file storage guide's §50)
        v
Every client editing "doc-42" connects to Server #7 SPECIFICALLY
        |
        v
Server #7 holds the ONE authoritative in-memory OT/CRDT session for "doc-42"
   (no other server needs to know or care about this document at all)
```

Routing every client of a **specific** document to the **same** server (rather than load-balancing connections arbitrarily) is what lets the OT engine's central-sequencing requirement (§17) hold true without needing cross-server coordination for every single keystroke — a direct application of [the file storage guide's consistent-hashing technique](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>), here routing documents to owning servers instead of routing object keys to storage nodes.

---

# 24. High-Level Architecture Diagram, Assembled

```text
                    Clients (browsers, each holding a local optimistic copy)
                                        |
                          Routing layer (consistent hashing by document ID, §23)
                                        |
        +---------------------+---------------------+---------------------+
        v                     v                     v                     v
  Collab Server #1      Collab Server #2      Collab Server #3      Collab Server #N
  (owns docs hashed      (owns docs hashed      (owns docs hashed      (owns docs hashed
   to this server)        to this server)        to this server)        to this server)
        |                     |                     |
        +---------------------+---------------------+
                                        |
                          Document store (durable content, §18, §38)
                          + Operation log (durability, §53)
```

Every box here was justified by a specific follow-up question (§12, §14, §19, §22) rather than assumed — exactly the standard [the JIRA guide's own architecture section](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already holds itself to.

---

# 25. Follow-up Question 7 — "Let's Get Concrete. Design the Class Model for Operational Transformation."

> **Interviewer:** *"High-level architecture is fine, but I want to see the actual algorithm. Show me how you'd represent an edit as data, and how you'd resolve two concurrent edits so every replica ends up with the same document."*

This is the pivot from architecture to algorithm — exactly [the JIRA guide's own equivalent moment](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) — and it's where OT's actual substance lives. §26–§30 build it in full, with one honest simplification stated upfront: real production OT systems (Google Wave's original protocol, ShareDB) represent an operation as a **composable sequence** of retain/insert/delete components, which handles every edge case generally; this guide uses single insert/delete operations with explicit position and length, which is simpler to reason about and correctly covers the common cases, at the cost of needing extra care for the rarer case of one operation's range fully overlapping another's (called out explicitly where it matters, §29).

---

# 26. Representing an Edit as an Operation

```java
// ot/Operation.java
public sealed interface Operation {
    int getBaseRevision(); // the document revision this operation was computed against — critical for §27
    String getClientId();  // used to break ties deterministically, §28

    record InsertOp(int position, String content, int baseRevision, String clientId) implements Operation {
        public int getBaseRevision() { return baseRevision; }
        public String getClientId() { return clientId; }
    }

    record DeleteOp(int position, int length, int baseRevision, String clientId) implements Operation {
        public int getBaseRevision() { return baseRevision; }
        public String getClientId() { return clientId; }
    }
}
```

`baseRevision` is what lets the server (§30) know exactly *which* other operations, if any, this operation needs to be transformed against — an operation computed against revision 5 that arrives after the document has already advanced to revision 7 must be transformed against whatever happened in revisions 6 and 7 before it can be safely applied.

---

# 27. The Transform Function: Reconciling Two Concurrent Operations

The core contract every transform function must satisfy: given two operations, `opA` and `opB`, **both computed against the same starting document state**, `transform(opA, opB)` produces a new version of `opA` — call it `opA'` — such that applying `opB` and then `opA'` produces the **exact same final document** as applying `opA` and then a correspondingly-transformed `opB'`. This is the property (Transformation Property 1, in the classic OT literature) that guarantees **convergence**: it doesn't matter which order the two concurrent operations actually reach a given replica, as long as each is correctly transformed against whatever already got applied first on that replica.

```java
public interface OperationTransformer {
    Operation transform(Operation opToTransform, Operation alreadyAppliedOp);
}
```

---

# 28. Implementing Insert-Insert Transformation

```java
public Operation.InsertOp transformInsertInsert(Operation.InsertOp a, Operation.InsertOp b) {
    if (a.position() < b.position()) {
        return a; // A's insert point is before B's — unaffected by B, no change needed
    }
    if (a.position() > b.position()) {
        // B's insert happened before A's position — A's position shifts right by however much B inserted
        return new Operation.InsertOp(a.position() + b.content().length(), a.content(), a.baseRevision(), a.clientId());
    }
    // EXACT same position — a genuine tie. Break it deterministically so EVERY replica resolves it
    // identically, regardless of which operation that replica happened to see "first."
    if (a.clientId().compareTo(b.clientId()) < 0) {
        return a; // A's client ID sorts first -> by convention, A's insert goes before B's -> no shift
    }
    return new Operation.InsertOp(a.position() + b.content().length(), a.content(), a.baseRevision(), a.clientId());
}
```

The tie-break on `clientId` is not an arbitrary detail — it's what makes the transform function **deterministic across every replica**. Without a consistent tie-break rule, two replicas could each "correctly" transform the same tied operations in opposite orders and diverge — a subtle bug that only manifests under exact-position concurrent edits, precisely the kind of edge case that makes OT implementations notoriously hard to get right without rigorous, systematic testing (§63).

---

# 29. Implementing Insert-Delete and Delete-Delete Transformation

```java
public Operation.InsertOp transformInsertAgainstDelete(Operation.InsertOp insert, Operation.DeleteOp delete) {
    if (insert.position() <= delete.position()) {
        return insert; // nothing before the insert point was deleted — unaffected
    }
    int deleteEnd = delete.position() + delete.length();
    if (insert.position() >= deleteEnd) {
        // the entire deleted range was before the insert point -> shift left by however much was removed
        return new Operation.InsertOp(insert.position() - delete.length(), insert.content(), insert.baseRevision(), insert.clientId());
    }
    // The insert point fell INSIDE the deleted range — clamp it to where the deletion happened.
    // (A more general retain/insert/delete representation, per §25's note, would preserve finer intent here;
    // clamping to the deletion point is the standard, defensible simplification most tutorials adopt.)
    return new Operation.InsertOp(delete.position(), insert.content(), insert.baseRevision(), insert.clientId());
}

public Operation.DeleteOp transformDeleteAgainstDelete(Operation.DeleteOp a, Operation.DeleteOp b) {
    int aEnd = a.position() + a.length();
    int bEnd = b.position() + b.length();
    if (aEnd <= b.position()) return a;                                    // A's range entirely before B's — unaffected
    if (a.position() >= bEnd) {
        return new Operation.DeleteOp(a.position() - b.length(), a.length(), a.baseRevision(), a.clientId()); // shift left
    }
    // Overlapping ranges: shrink A by whatever portion B already deleted, to avoid double-deleting
    // characters that no longer exist. This is exactly the trickiest case §25 flagged.
    int overlapStart = Math.max(a.position(), b.position());
    int overlapEnd = Math.min(aEnd, bEnd);
    int overlapLength = Math.max(0, overlapEnd - overlapStart);
    int newPosition = a.position() >= b.position() ? b.position() : a.position();
    int newLength = a.length() - overlapLength;
    return new Operation.DeleteOp(newPosition, Math.max(0, newLength), a.baseRevision(), a.clientId());
}
```

The delete-delete overlap case is exactly where §25's simplification shows its seams — shrinking a single `(position, length)` range correctly handles the common cases (no overlap, one fully containing the other, partial overlap at one end) but a delete that's split into two disjoint remaining pieces by an already-applied delete in its *middle* genuinely cannot be represented by one `DeleteOp` anymore. A production system's retain/insert/delete component sequence (§25) sidesteps this entirely by representing the result as multiple components instead of forcing it back into one operation — worth stating explicitly as a real limitation of this guide's simplified model, not glossed over.

---

# 30. The Central Server's Role: Sequencing and Broadcasting Transformed Operations

```java
// ot/OtDocumentSession.java
public class OtDocumentSession {
    private String documentContent;
    private int currentRevision = 0;
    private final List<Operation> appliedOperationsLog = new ArrayList<>(); // indexed by revision number
    private final OperationTransformer transformer = new OperationTransformer();

    public synchronized Operation receiveOperation(Operation incoming) {
        Operation transformed = incoming;
        // Transform the incoming op against every operation that was applied AFTER the client's baseRevision —
        // i.e., everything this client didn't know about yet when it computed its own operation.
        for (int rev = incoming.getBaseRevision(); rev < currentRevision; rev++) {
            transformed = transformer.transform(transformed, appliedOperationsLog.get(rev));
        }
        documentContent = applyToContent(documentContent, transformed);
        appliedOperationsLog.add(transformed);
        currentRevision++;
        broadcastToOtherClients(transformed); // §15 — every other connected client receives this SAME transformed op
        return transformed;
    }
    // applyToContent(...) omitted for brevity — a straightforward string splice based on the operation's type
}
```

The `synchronized` keyword here is doing real, load-bearing work: operations for one document must be transformed and applied **one at a time, in a single, agreed-upon order** — this is precisely the "needs a central sequencer" property §17's comparison table names as OT's defining requirement, and it's exactly why §23's session-affinity routing (one server owns one document's session) exists — this method must never run concurrently for the same document across two different server instances.

---

# 31. Class Diagram: The OT Engine

```text
┌──────────────────┐  receiveOperation()   ┌──────────────────────┐
│  OtDocumentSession │──────────────────────>│ OperationTransformer  │
├──────────────────┤                        ├──────────────────────┤
│ documentContent   │                        │ + transform(a, b)     │
│ currentRevision   │                        └──────────┬───────────┘
│ appliedOpsLog[]   │                                   │ operates on
└──────────────────┘                                   v
         ^                                    ┌──────────────────┐
         │ appends to log,                    │    Operation      │ (sealed interface)
         │ broadcasts result                  ├──────────────────┤
         │                                     │ baseRevision      │
┌──────────────────┐                          │ clientId          │
│  Connected Client │ (one per active editor)  └────────┬─────────┘
│  local optimistic  │                                   │ implements
│  copy + pending    │                        ┌──────────┴──────────┐
│  operation queue    │                        v                     v
└──────────────────┘                 ┌──────────────────┐  ┌──────────────────┐
                                      │    InsertOp       │  │    DeleteOp       │
                                      ├──────────────────┤  ├──────────────────┤
                                      │ position, content │  │ position, length  │
                                      └──────────────────┘  └──────────────────┘
```

---

# 32. Follow-up Question 8 — "OT Requires a Central Server to Sequence Operations. What If You Don't Want That?"

> **Interviewer:** *"You said OT needs one server to be the source of truth for ordering. What if I want this to work peer-to-peer, or I want any replica — including an offline one — to be able to apply operations in whatever order they happen to arrive, and still guarantee everyone converges?"*

This is the exact motivation for **CRDTs** (Conflict-free Replicated Data Types) — a fundamentally different strategy that sidesteps transformation entirely by making the underlying data structure itself immune to operation ordering.

---

# 33. CRDT Fundamentals: Convergence Without Coordination

A CRDT's defining property: applying a set of operations **in any order** (not just the order they were generated) produces the **same final state** — this is achieved by designing every operation to be **commutative** (order doesn't matter) and **idempotent** (applying the same operation twice has no additional effect beyond applying it once). Where OT achieves convergence by actively *transforming* operations to account for ordering, a CRDT achieves it by making ordering **irrelevant** to the result in the first place — a structurally different, not just implementationally different, solution to §10's original problem.

---

# 34. A Simple CRDT for Text: The RGA (Replicated Growable Array) Idea

The key trick: instead of positioning a character by a numeric index (which, as §10 showed, drifts as other edits land), give every character a **globally unique, immutable identifier**, and record what it was inserted **immediately after** — its "origin." The document's actual visible order is then always computed by **walking this chain of origin references**, never by trusting a raw numeric position at all.

```text
Original: "cat"  with characters identified as  c(id=1) -> a(id=2) -> t(id=3)

Alice inserts 'h' after nothing (at the start):        h(id=4, origin=START)
Bob concurrently inserts 's' after t (id=3):            s(id=5, origin=3)

Both operations reference STABLE ids ("origin=3"), never a numeric position that could drift —
applying them in EITHER order reconstructs the same chain: START -> h(4) -> c(1) -> a(2) -> t(3) -> s(5)
                                                          = "hcats" -- exactly the correct converged result from §10
```

---

# 35. Implementing a Character-Level CRDT

```java
// crdt/CharacterId.java — a globally unique, totally-ordered identifier for one character
public record CharacterId(long counter, String siteId) implements Comparable<CharacterId> {
    @Override
    public int compareTo(CharacterId other) {
        int byCounter = Long.compare(counter, other.counter);
        return byCounter != 0 ? byCounter : siteId.compareTo(other.siteId); // deterministic tie-break, same idea as §28
    }
}

// crdt/CrdtCharacter.java
public class CrdtCharacter {
    final CharacterId id;
    final char value;
    final CharacterId originLeft; // null = inserted at the very start of the document
    boolean deleted = false;      // a TOMBSTONE, never physically removed — see the note below

    CrdtCharacter(CharacterId id, char value, CharacterId originLeft) {
        this.id = id; this.value = value; this.originLeft = originLeft;
    }
}

// crdt/CrdtDocument.java
public class CrdtDocument {
    private final List<CrdtCharacter> characters = new ArrayList<>(); // kept in CONVERGED visible order

    public CrdtCharacter insertAfter(CharacterId originLeft, char value, CharacterId newId) {
        CrdtCharacter newChar = new CrdtCharacter(newId, value, originLeft);
        int insertIndex = findInsertIndex(originLeft, newId);
        characters.add(insertIndex, newChar);
        return newChar; // this object IS the operation — broadcast it as-is to every other replica
    }

    private int findInsertIndex(CharacterId originLeft, CharacterId newId) {
        int originIndex = (originLeft == null) ? -1 : indexOfId(originLeft);
        int i = originIndex + 1;
        // Skip past any characters ALREADY inserted at this same origin with a HIGHER id (§28's same tie-break
        // idea, applied here structurally instead of via an explicit transform function) — this is what makes
        // insertion order deterministic across replicas regardless of which operation arrives first.
        while (i < characters.size() && characters.get(i).originLeft != null
               && characters.get(i).originLeft.equals(originLeft)
               && characters.get(i).id.compareTo(newId) > 0) {
            i++;
        }
        return i;
    }

    public void markDeleted(CharacterId targetId) {
        indexOfCharacter(targetId).deleted = true; // NEVER physically removed — see §35's tombstone note
    }

    public String toVisibleString() {
        StringBuilder sb = new StringBuilder();
        for (CrdtCharacter c : characters) if (!c.deleted) sb.append(c.value);
        return sb.toString();
    }
    // indexOfId(...) / indexOfCharacter(...) omitted for brevity — linear scans by CharacterId equality
}
```

**Why tombstones instead of physically deleting a character?** If character `t(id=3)` were physically removed the instant it's deleted, and another replica concurrently receives an insert with `origin=3` (an insert that happened *after* `t` before the deletion was known), that replica would have no way to resolve "insert after a character that no longer exists" — keeping the deleted character as an invisible tombstone (excluded only from `toVisibleString()`) preserves it as a stable anchor point for any concurrent operation that still refers to it, at the honest, accepted cost of the document's internal representation growing forever with every deletion, never shrinking (§66 names a real mitigation — periodic tombstone garbage collection once every replica has acknowledged an operation).

---

# 36. OT vs CRDT: A Direct Comparison

| | Operational Transformation | CRDTs |
|---|---|---|
| Requires central sequencing | Yes (§30) | No — any replica can apply operations in any order |
| Memory overhead | Low — operations don't accumulate in the document itself | Higher — tombstones (§35) accumulate permanently without periodic garbage collection |
| Algorithm complexity | Transform functions have many hand-derived cases, historically a rich source of subtle bugs (§29) | The core structure (unique IDs + origin references) is conceptually simpler, though correctly implementing the ordering/tie-break logic still requires care |
| Enables true peer-to-peer / offline-first | Not naturally — needs a server or an agreed total order | Yes — this is CRDTs' headline capability |
| Best fit | A product already built around a central server, prioritizing lower memory overhead | A product needing offline-first or peer-to-peer collaboration without any always-available central authority |

Neither is a strictly better choice — Google Docs' long-standing OT-based architecture and Figma's/Yjs-based tools' CRDT architecture are both large-scale, successful, production systems built on genuinely different answers to the exact same underlying problem, chosen for reasons specific to each product's own constraints.

---

# 37. Class Diagram: The CRDT Engine

```text
┌──────────────────┐  1        *  ┌──────────────────┐
│   CrdtDocument     │─────────────>│  CrdtCharacter     │
├──────────────────┤   holds, in   ├──────────────────┤
│ characters[]       │   converged   │ id: CharacterId   │
│                     │   visible     │ value: char       │
│ + insertAfter(...)  │   order       │ originLeft: Id    │
│ + markDeleted(...)  │               │ deleted: boolean  │ (tombstone, §35)
│ + toVisibleString() │               └────────┬─────────┘
└──────────────────┘                          │ identified by
                                                v
                                       ┌──────────────────┐
                                       │   CharacterId      │
                                       ├──────────────────┤
                                       │ counter: long      │
                                       │ siteId: String      │
                                       │ (Comparable — total │
                                       │  order tie-break)   │
                                       └──────────────────┘
```

---

# 38. The Document Entity and Version History

```java
// document/Document.java
public class Document {
    private final String id;
    private final String ownerId;
    private String title;
    private String currentContent;      // the durable, persisted content (§18) — kept in sync with the live session
    private int currentRevision;         // matches OtDocumentSession.currentRevision (§30), or a CRDT equivalent
    private final Instant createdAt;
    private Instant lastModifiedAt;
}
```

`currentContent` and `currentRevision` are deliberately kept in the durable store **in addition to** the live in-memory session state (§30) — the in-memory session is fast but ephemeral (gone the instant the owning server restarts); the durable copy is what a fresh session initializes from (§18) and what a completely disconnected viewer's read request is served from without needing to spin up a live session at all.

---

# 39. The Session/Connection Model

```java
// session/EditSession.java — one per document CURRENTLY being actively edited
public class EditSession {
    private final String documentId;
    private final Map<String, WebSocketConnection> connectedClients = new ConcurrentHashMap<>(); // keyed by clientId
    private final OtDocumentSession otSession; // or a CrdtDocument, depending on which engine this deployment uses

    public void broadcast(Operation resolvedOp, String excludeClientId) {
        for (var entry : connectedClients.entrySet()) {
            if (!entry.getKey().equals(excludeClientId)) entry.getValue().send(resolvedOp); // §30's fan-out, concretely
        }
    }
}
```

An `EditSession` exists only while **at least one** client is connected to that document — created lazily on the first connection, torn down (after persisting final state to `Document`, §38) once the last client disconnects, exactly the "session affinity" resource this guide's §23 routing layer is directing traffic toward.

---

# 40. Follow-up Question 9 — "How Do You Implement Live Cursors and Selections?"

> **Interviewer:** *"Beyond the text itself, users need to see where everyone else's cursor is, live. How do you build that without it interfering with the actual document synchronization you just designed?"*

Exactly §21's point, now made concrete: cursor positions are broadcast through the **same** `EditSession`/WebSocket infrastructure (§39) as content operations, but as a **completely separate message type** that never touches the OT/CRDT engine (§30, §35) at all — a cursor "move" is not a document mutation, has no conflict to resolve, and simply overwrites whatever position was last known for that user.

---

# 41. Presence and Awareness

```java
// session/PresenceTracker.java
public class PresenceTracker {
    private final Map<String, CursorPosition> currentPositions = new ConcurrentHashMap<>(); // clientId -> latest position

    public void updateCursor(String clientId, CursorPosition position) {
        currentPositions.put(clientId, position); // last-write-wins — perfectly correct for ephemeral presence, unlike content!
        broadcastPresenceUpdate(clientId, position);
    }

    public void onClientDisconnect(String clientId) {
        currentPositions.remove(clientId);
        broadcastPresenceUpdate(clientId, null); // null signals "this user's cursor is no longer visible"
    }
}
```

Note the comment: **last-write-wins is completely correct here**, in stark contrast to §10's insistence that last-write-wins is wrong for document content — because presence has no notion of "losing" information a user cares about (an overwritten cursor position was never something anyone needed to preserve), while a dropped keystroke absolutely is. The same technique that would be a serious bug for content (§9) is the exact right tool for presence — a clean illustration that "which conflict-resolution strategy is correct" depends entirely on what's actually at stake in the data being resolved.

---

# 42. Class Diagram: Document, Session, and Presence

```text
┌──────────────┐   1        1   ┌──────────────────┐   1        1   ┌──────────────────┐
│   Document    │───────────────>│   EditSession      │───────────────>│  OtDocumentSession│
├──────────────┤   backs (while  ├──────────────────┤   delegates to  │  or CrdtDocument   │
│ id           │   active)       │ connectedClients{} │                 └──────────────────┘
│ currentContent│                │ otSession/crdt      │
│ currentRevision│               └────────┬───────────┘
└──────────────┘                          │ 1
                                            │
                                            v *
                                   ┌──────────────────┐
                                   │ PresenceTracker    │
                                   ├──────────────────┤
                                   │ currentPositions{} │  (clientId -> CursorPosition, last-write-wins, §41)
                                   └──────────────────┘
```

---

# 43. Follow-up Question 10 — "How Do You Handle Sharing and Permissions (Viewer, Commenter, Editor)?"

> **Interviewer:** *"Not everyone who opens this document should be able to type in it. How does your design enforce Viewer vs. Commenter vs. Editor access?"*

Exactly [the JIRA guide's own per-project RBAC design](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>), applied here at the per-document level instead of per-project — the same underlying pattern (a role, scoped to a specific resource, checked before any mutating action), because the requirement ("different users, different permission levels, on the same shared resource") is structurally identical.

---

# 44. Role-Based Sharing Model

```java
public enum ShareRole { VIEWER, COMMENTER, EDITOR }

public class DocumentAccess {
    private final String documentId;
    private final String userId;
    private ShareRole role;
}

public class AccessChecker {
    public void requireEditAccess(String userId, String documentId) {
        DocumentAccess access = accessRepository.find(userId, documentId);
        if (access == null || access.getRole() != ShareRole.EDITOR) {
            throw new AccessDeniedException(userId, documentId);
        }
    }
}
```

`AccessChecker.requireEditAccess` is called **before** an incoming `Operation` (§26) is ever handed to `OtDocumentSession.receiveOperation` (§30) — exactly the same "gate the write before the domain logic runs" ordering [the JIRA guide's `PermissionChecker`](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already established, ensuring a Viewer's WebSocket connection can receive broadcasts (read-only) but can never successfully submit an operation that mutates the document.

---

# 45. Class Diagram: Sharing and Access Control

```text
┌──────────┐   *        *   ┌────────────────────┐   *        1   ┌──────────┐
│   User    │───────────────>│  DocumentAccess      │<───────────────│ Document │
├──────────┤                 ├────────────────────┤                 ├──────────┤
│ id       │                 │ userId              │                 │ id       │
└──────────┘                 │ documentId           │                 │ ownerId  │
                              │ role: ShareRole      │                 └──────────┘
                              └────────────────────┘
                                        |
                                        v
                              ┌────────────────────┐
                              │     ShareRole        │ (enum: VIEWER, COMMENTER, EDITOR)
                              └────────────────────┘
```

---

# 46. Follow-up Question 11 — "How Do You Implement Undo/Redo in a Multi-User Environment?"

> **Interviewer:** *"Alice types 'hello', then Bob inserts a word in the middle of it, then Alice hits undo. What should happen — and what would naively break if you just reverted Alice's last known document state?"*

Naively snapshotting "the document before Alice's last edit" and restoring it would **silently discard Bob's edit** — exactly the kind of lost-update problem this entire guide exists to prevent, now reappearing inside undo/redo specifically. The correct approach: undo isn't "restore an old snapshot," it's "generate and apply a **new** operation that reverses the effect of one specific past operation" — which composes correctly with everything that's happened since, including other users' edits.

---

# 47. Undo/Redo in a Collaborative Context

```java
// undo/UndoManager.java
public class UndoManager {
    private final Deque<Operation> localUndoStack = new ArrayDeque<>(); // only THIS client's own operations
    private final Deque<Operation> localRedoStack = new ArrayDeque<>();

    public void recordLocalOperation(Operation op) {
        localUndoStack.push(op);
        localRedoStack.clear(); // a new edit invalidates any pending redo, standard editor convention
    }

    public Operation undo() {
        Operation lastOp = localUndoStack.pop();
        Operation inverse = invert(lastOp); // e.g. InsertOp(pos, "abc") -> DeleteOp(pos, length=3)
        localRedoStack.push(lastOp);
        return inverse; // submitted through the SAME receiveOperation/transform pipeline (§30) as any other edit
    }

    private Operation invert(Operation op) {
        return switch (op) {
            case Operation.InsertOp i -> new Operation.DeleteOp(i.position(), i.content().length(), i.baseRevision(), i.clientId());
            case Operation.DeleteOp d -> new Operation.InsertOp(d.position(), d.deletedContent(), d.baseRevision(), d.clientId());
        };
    }
}
```

Submitting the inverse operation through the **exact same** `receiveOperation` pipeline (§30) as any ordinary edit is the key design decision: undo gets §27–§29's transformation logic **for free** — if Bob inserted text in the middle of what Alice is now undoing, the inverse delete operation is transformed against Bob's insert exactly like any other concurrent pair of operations, correctly adjusting *where* the undo applies rather than blindly reverting to stale positions. Each client's undo stack is also deliberately **local, per-client** — undoing only ever reverses *that specific user's own* past actions, never another user's, which matches how every real collaborative editor's undo behaves.

---

# 48. Follow-up Question 12 — "How Do You Show Document History / Version Snapshots?"

> **Interviewer:** *"A user wants to see what the document looked like an hour ago, and restore that version. You have a stream of individual operations — how do you turn that into browsable 'versions'?"*

Replaying every operation from the beginning of the document's history, every time someone wants to view an old version, works but grows more expensive the older the document gets — exactly [TinyDB's own crash-recovery cost problem](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>), and solved the same way: periodic **snapshots**.

---

# 49. Snapshotting and Version History

```java
// document/VersionHistory.java
public class VersionHistory {
    public record Snapshot(int revision, String content, Instant takenAt, String label) { }

    private final List<Snapshot> snapshots = new ArrayList<>();
    private final List<Operation> operationLogSinceLastSnapshot = new ArrayList<>();

    public void maybeSnapshot(int currentRevision, String currentContent) {
        if (currentRevision % 100 == 0) { // a snapshot every 100 operations — the same bound-replay-cost idea as TinyDB's §13
            snapshots.add(new Snapshot(currentRevision, currentContent, Instant.now(), null));
            operationLogSinceLastSnapshot.clear();
        }
    }

    public String reconstructAsOf(int targetRevision) {
        Snapshot nearestSnapshot = snapshots.stream()
            .filter(s -> s.revision() <= targetRevision)
            .max(Comparator.comparingInt(Snapshot::revision))
            .orElseThrow();
        // replay ONLY the operations between nearestSnapshot.revision() and targetRevision — never from the start
        return replayFrom(nearestSnapshot.content(), nearestSnapshot.revision(), targetRevision);
    }
    // replayFrom(...) omitted for brevity — applies each logged operation in sequence, same mechanism as §30
}
```

This is precisely the "snapshot plus replay only the remaining log" pattern [TinyDB's crash recovery](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) and [the file storage guide's GFS/HDFS NameNode checkpointing](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) both already use — the same idea, appearing a third time in this guide series, because "bound the cost of reconstructing history" is a genuinely recurring problem with one genuinely recurring solution.

---

# 50. Follow-up Question 13 — "What Guarantees Does Your System Actually Provide? Is It Strongly Consistent?"

> **Interviewer:** *"Be precise. Is this system strongly consistent? What exact guarantee can you state and defend?"*

No — and saying so plainly is the correct, stronger answer than overclaiming. The precise, defensible guarantee is **eventual consistency with guaranteed convergence**: at any given instant, different users' screens may show momentarily different content (Alice's screen already reflects her own just-typed keystroke; Bob's hasn't received it yet), but **once all in-flight operations have been delivered and applied, every replica converges to the identical final document** — the mathematical property §27 (OT) and §33 (CRDTs) are each specifically designed to guarantee. This is a different, weaker-sounding but entirely appropriate guarantee compared to a database transaction's strong consistency — and knowing the difference, and why the weaker guarantee is the *correct* choice for this specific problem, is exactly the signal this question is probing for.

---

# 51. Eventual Consistency and Convergence Guarantees

Stating the guarantee even more precisely, in terms already built:

- **Convergence**: for OT, guaranteed by Transformation Property 1 (§27) holding for every pair of operations the transform functions handle (§28–§29, with §29's honestly-named limitation). For CRDTs, guaranteed structurally by commutative, idempotent operations over unique, stable identifiers (§33–§35).
- **Intention preservation**: a user's edit should have the *effect they intended*, not just *some* effect that happens to converge — this is a real, separate property from convergence alone (an operation could converge to a valid document while still landing somewhere the user never meant), and it's exactly why §29's insert-into-a-deleted-range clamping and §47's undo-via-inversion are designed the way they are, rather than any arbitrary convergent resolution being considered acceptable.

---

# 52. Follow-up Question 14 — "What Happens If the Server Crashes Mid-Broadcast?"

> **Interviewer:** *"The Collaboration Server receives Alice's operation, applies it, and crashes before finishing broadcasting it to Bob and Carol. What state is everyone in now, and how do you recover?"*

This is the exact same class of problem [TinyDB's crash-recovery sections](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) solve for database writes, now applied to collaborative operations: the fix is **durability before acknowledgment** — an operation must be durably logged (§53) *before* the server tells the originating client it succeeded, so a crash can never lose an operation the client believes was accepted.

---

# 53. Server Crash Recovery and Operation Durability

```java
public Operation receiveOperationDurably(Operation incoming) throws IOException {
    Operation transformed = transformAgainstHistory(incoming); // §30
    operationLog.append(transformed); // durable write-ahead log entry, BEFORE anything else — exactly TinyDB's §9 discipline
    documentContent = applyToContent(documentContent, transformed);
    broadcastToOtherClients(transformed); // only AFTER durability is confirmed
    return transformed;
}

// On server restart, for any document that had an active session:
public void recoverSession(String documentId) throws IOException {
    Document durableDoc = documentRepository.find(documentId); // last confirmed snapshot (§49)
    List<Operation> unappliedOps = operationLog.readSince(durableDoc.getCurrentRevision());
    String reconstructed = durableDoc.getCurrentContent();
    for (Operation op : unappliedOps) reconstructed = applyToContent(reconstructed, op);
    // reconstructed is now EXACTLY the state the crashed server had right before it died — no operation lost
}
```

If the server crashes *after* `operationLog.append` but *before* `broadcastToOtherClients`, recovery replays the logged operation and re-broadcasts it — Bob and Carol simply receive it a little late, with no data loss. If it crashes *before* the append call completes, the operation was never durably accepted in the first place, and the client's own local retry/reconnect logic (§20, reused directly here) resubmits it, exactly as if it had been offline. There is **no window** in this design where an operation is acknowledged to a client and then lost — the same non-negotiable durability property [TinyDB's WAL](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) provides for database writes.

---

# 54. Follow-up Question 15 — "10 Million Documents, 500K Concurrent Editors. What Breaks First?"

> **Interviewer:** *"The product is a hit. 10 million documents exist; 500,000 users are actively editing at any given moment, spread across maybe 200,000 concurrently-open documents. Walk me through, specifically, what breaks first and what you do about it."*

The same style of question [the JIRA guide's own scaling follow-up](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) asks — testing sequencing and root-cause reasoning, not a generic checklist.

---

# 55. Sharding Documents Across Servers

§23's session-affinity routing already shards by document — at this scale, the question becomes whether **one** consistent-hash ring across a fixed server pool is enough, or whether a **coordinator** is needed to actively rebalance hot documents (a viral, thousand-viewer document, even within this guide's "dozens of concurrent editors" assumption, could still dominate one server's resources) onto dedicated capacity — the same "some keys are hotter than others" problem [the file storage guide's consistent hashing discussion](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) acknowledges rather than assumes away.

---

# 56. Horizontal Scaling of the Real-Time Layer (Pub/Sub Backbone)

Each `EditSession` (§39) is currently assumed to live entirely within one server's memory — correct for §23's model, but it means that server's WebSocket connection count and CPU (running transform functions, §27–§29) both scale with how many *hot* documents happen to hash to it. Horizontally scaling this layer means running many stateless-at-the-routing-level Collaboration Server instances behind the consistent-hash router (§23), with the **document store and operation log** (§18, §53) — not any single server's memory — remaining the actual, durable source of truth every server instance can rebuild a session from after a restart or a rebalance.

---

# 57. Rate Limiting and Backpressure

A buggy client (or a malicious one) sending an unbounded stream of operations can overwhelm one document's session — a **per-client rate limit** on operations-per-second, enforced at the WebSocket connection level before an operation ever reaches `receiveOperationDurably` (§53), protects the shared session (and every other legitimate editor on that same document) from one misbehaving connection, the same "protect shared infrastructure from one bad actor" instinct [the JIRA guide's own rate-limiting section](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already establishes at the API-gateway level.

---

# 58. Full Worked Example: Two Users Editing the Same Document

```text
Alice and Bob both connected to "doc-42", currentRevision = 5, content = "cat"

Alice types 'h' at position 0    -> InsertOp(pos=0, "h", baseRevision=5, client=Alice)
Bob types 's' at position 3      -> InsertOp(pos=3, "s", baseRevision=5, client=Bob)    (concurrent, same base revision)

Both sent to the owning server (§23) at nearly the same instant:

1. Server receives Alice's op FIRST:
   -> AccessChecker.requireEditAccess(alice, "doc-42")                          (§44)
   -> transformAgainstHistory: no ops since revision 5 yet -> unchanged
   -> operationLog.append(...)                                                   (§53, durable BEFORE anything else)
   -> apply -> content becomes "hcat", currentRevision = 6
   -> broadcast to Bob

2. Server receives Bob's op SECOND (still based on revision 5, since Bob hasn't seen Alice's op yet):
   -> AccessChecker.requireEditAccess(bob, "doc-42")                             (§44)
   -> transformAgainstHistory: transform against Alice's now-applied InsertOp    (§28)
        Bob's position 3 > Alice's position 0 -> shift right by 1 -> position becomes 4
   -> operationLog.append(...)                                                   (§53)
   -> apply transformed op -> content becomes "hcats", currentRevision = 7
   -> broadcast to Alice

Final state, EVERYWHERE: "hcats" -- exactly §10's originally-stated correct answer, now actually derived.
```

Every mechanism this guide built — permission checks, durability-before-broadcast, and the transform function itself — appears in this one worked trace, resolving the exact scenario §10 posed as the guide's opening motivating problem.

---

# 59. Final Architecture

```text
                    Clients (optimistic local edits, §13)
                              |
              Routing layer (consistent hashing by document ID, §23)
                              |
        +---------------------+---------------------+
        v                     v                     v
  Collab Server         Collab Server         Collab Server
  EditSession (§39) --- OtDocumentSession/CrdtDocument (§30, §35)
       |                     |
  AccessChecker (§44) gates every incoming operation
       |
  operationLog.append() BEFORE broadcast (§53, durability)
       |
        +---------------------+---------------------+
        v                                           v
  Document store (§38)                    PresenceTracker (§41, separate from content)
  + VersionHistory snapshots (§49)
```

---

# 60. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Command** | Each `Operation` is a self-contained, serializable description of an action, invertible for undo (§47) | §26, §47 |
| **Strategy** | Swapping the conflict-resolution engine (OT vs CRDT) behind the same `EditSession` abstraction | §17, §39 |
| **Memento** | Version snapshots capturing document state for later restoration | §49 |
| **Observer** (implicit) | Broadcasting a resolved operation to every connected client, decoupled from who's actually listening | §30, §39 |
| **Consistent Hashing** | Routing a document's editors to its one owning server | §23, reused directly from [the file storage guide's §50](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) |

---

# 61. SOLID Principles Applied

| Principle | Where it holds |
|---|---|
| **S**ingle Responsibility | `OperationTransformer` only transforms; `AccessChecker` only checks permissions; `PresenceTracker` only tracks cursors — none conflated |
| **O**pen/Closed | Adding a new operation type (e.g. a `FormatOp` for rich-text styling) means implementing the `Operation` interface and its transform cases — existing operation handling is untouched |
| **L**iskov Substitution | Any `Operation` subtype is interchangeable wherever the sealed interface is accepted (logging, broadcasting, transforming) |
| **I**nterface Segregation | `OperationTransformer` exposes exactly one method — no implementation is forced to support unrelated behavior |
| **D**ependency Inversion | `EditSession` depends on the *idea* of a conflict-resolution engine, not a hard-coded OT or CRDT implementation — §17's comparison is possible precisely because both sit behind the same seam |

---

# 62. Common Mistakes

- **Mistake 1 — Using raw numeric positions without transformation (§10).** The exact bug this entire guide exists to prevent — naive position-based application produces `"hcast"` instead of `"hcats"`.
- **Mistake 2 — No deterministic tie-break for same-position concurrent inserts (§28).** Without one, different replicas can transform identically-positioned operations in different orders and diverge — a bug that only appears under exact-timing concurrent edits, making it notoriously hard to catch without dedicated testing (§63).
- **Mistake 3 — Physically deleting CRDT characters instead of tombstoning them (§35).** Breaks any concurrent operation still referencing the deleted character as an anchor point.
- **Mistake 4 — Routing presence updates through the same conflict-resolution pipeline as content (§41).** Unnecessary complexity and latency for data that never needed convergence guarantees in the first place.
- **Mistake 5 — Acknowledging an operation to the client before it's durably logged (§53).** Reopens exactly the "acknowledged then lost" gap a WAL-style discipline exists to close.
- **Mistake 6 — Implementing undo as "restore an old document snapshot" instead of "apply an inverse operation" (§46-§47).** Silently discards every other user's edits made since that snapshot.

---

# 63. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| OT transform functions (§28-§29) | Applying `(opA then transform(opB,opA))` and `(opB then transform(opA,opB))` produce IDENTICAL final strings, for many random pairs of operations | Property-based/randomized testing — this is the standard technique real OT libraries use, precisely because hand-picked example cases rarely find the subtle bugs (§62's Mistake 2) |
| CRDT convergence (§34-§35) | Applying the same set of operations in every possible order always converges to the same `toVisibleString()` result | Generate random operation sets, apply in shuffled orders across simulated replicas, assert identical results |
| Durability (§53) | An operation is recoverable after a simulated crash between logging and broadcasting | Kill the process (or simulate it) at each point in `receiveOperationDurably`, restart, assert no operation is lost |
| Access control (§44) | A Viewer's operation is rejected before ever reaching the transform engine | Attempt an edit as a Viewer-role user, assert `AccessDeniedException` and assert the document content is unchanged |
| Undo/redo (§47) | Undoing one user's operation correctly transforms against another user's interleaved edit | The exact scenario in §46's motivating question, as an automated test |

The property-based OT test is the single most valuable one in this list — hand-written example-based tests reliably miss the exact class of tie-break and boundary bugs (§62) that make OT implementations infamous for subtle, hard-to-reproduce divergence bugs in production.

---

# 64. Suggested Future Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Rich text / structured formatting | Bold, italic, headings, embedded images/tables, not just plain character strings | A richer `Operation` type set (§26) covering formatting spans, transformed with the same core principles but more cases |
| Tombstone garbage collection | Bounding the CRDT's unbounded memory growth (§35) | Periodically compact tombstones once every active replica has acknowledged past a given revision — the same "safe once nobody still needs it" logic [TinyDB's compaction](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) already uses for superseded log entries |
| Comment threads anchored to content | Comments that "stick" to their original text even as the document is edited around them | Anchoring a comment to a stable CRDT character ID (§34) or an OT position tracked through every subsequent transform, rather than a raw, driftable numeric offset |
| Peer-to-peer sync (true offline-first, no server round trip needed between peers) | Direct device-to-device collaboration without a central server in the loop | Builds directly on the CRDT engine (§32-§37), which was specifically designed not to require central sequencing |
| Fine-grained, field-level permissions on structured documents (e.g. a spreadsheet) | "This user can edit column A but not column B" | Extends §44's `ShareRole` to be range- or region-aware, checked per-operation rather than per-document |
| AI-assisted conflict summarization | Surfacing "here's what changed while you were away" in plain language, not just a raw diff | A consumer of the same operation log (§53) already durably capturing every change |

---

# 65. Progressive Interview Question Set

**Level 1 — The core problem**
1. Why is rejecting a concurrent edit (as optimistic locking does for a database row) not an acceptable solution here?
2. Walk through §10's `"cat"` example by hand and explain exactly why naive position-based application produces the wrong result.

**Level 2 — Operational Transformation**
3. State Transformation Property 1 precisely, and explain what it guarantees.
4. Why does the insert-insert transform need a deterministic tie-break, and what specifically goes wrong without one?
5. Why does OT require a central sequencer, mechanically — what part of the algorithm actually depends on it?

**Level 3 — CRDTs**
6. Explain why CRDT deletions use tombstones instead of physically removing data.
7. Compare OT and CRDTs on memory overhead and coordination requirements, with concrete justification for each.

**Level 4 — Architecture**
8. Why is presence (cursors) handled through a separate mechanism from content synchronization?
9. Explain session affinity/consistent hashing's role in this design, and what would break without it.

**Level 5 — Correctness under failure**
10. State the exact consistency guarantee this system provides, precisely, and justify why it's the right one for this problem.
11. Walk through what happens if the server crashes between receiving an operation and broadcasting it.

**Final challenge:** Design collaborative editing for a **spreadsheet** rather than plain text — cells are addressed by (row, column), formulas can reference other cells, and a formula's displayed value depends on cells that might be concurrently edited by someone else. What has to change about the operation model (§26) and the transform/convergence logic (§27-§37), and what entirely new class of conflict (beyond position drift) does a formula dependency introduce that plain text editing never had to confront?

---

# 66. Final Takeaway

Every mechanism in this guide traces back to one refusal stated in §9: **a collaborative editor must never reject or silently discard a user's keystroke just because someone else typed at the same moment.** Operational Transformation honors that refusal by actively recomputing an operation's meaning against whatever already happened (§27-§30); CRDTs honor the identical refusal by designing data that never needs recomputing in the first place, because its identity was never tied to a driftable position to begin with (§33-§35). Presence, undo, version history, sharing, and crash recovery are not separate features bolted onto this core — each one is the *same* underlying idea (merge, don't reject; converge, don't overwrite) applied to a different slice of the problem: presence chooses last-write-wins deliberately because nothing is actually at stake there (§41); undo submits an inverse operation through the identical transform pipeline rather than reverting a snapshot (§47); crash recovery guarantees durability before acknowledgment for the identical reason a database's WAL does (§53). Understanding *why* each of these choices follows necessarily from §9's opening refusal — not memorizing them as a checklist — is what turns "I know Google Docs uses something called Operational Transformation" into an actual system-design answer.


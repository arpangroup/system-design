# Real-Time Collaboration Tool — System Design Guide

> **Interview level:** Senior / Staff Software Engineer  
> **Difficulty:** Advanced distributed systems  
> **Primary example:** Google Docs / Notion / Figma-style collaborative workspace  
> **Implementation preference:** Java 21 + Spring Boot for backend, React for web client  
> **Design principles:** SOLID, clean architecture, extensibility, fault tolerance

---

# 1. System Design Interview Question

## Problem Statement

> **Design a real-time collaboration platform that allows multiple users to work on the same document simultaneously.**
>
> Users should be able to edit documents concurrently, see other users' changes in near real time, see presence/cursors, comment on content, and continue working despite temporary network failures.
>
> The system should support millions of users and documents while maintaining consistency for concurrent edits.

Examples of products with related characteristics:

- Google Docs
- Microsoft 365 collaborative editing
- Notion
- Figma
- Miro

We will design a generic platform rather than copying any particular product.

---

# 2. Interview Follow-Ups

A good interviewer can progressively increase the difficulty.

## Follow-up 1 — Basic collaboration

- Can two users edit the same document?
- How are changes propagated?
- What happens when both edit simultaneously?

## Follow-up 2 — Ordering

- How do we guarantee a consistent operation order?
- What happens if messages arrive out of order?

## Follow-up 3 — Offline editing

- What happens when a user loses connectivity?
- How are offline operations merged?

## Follow-up 4 — Presence

- How do users see who is online?
- How do we display cursors/selections?

## Follow-up 5 — Scale

- How do we support millions of concurrent connections?
- How do we distribute WebSocket connections?

## Follow-up 6 — Persistence

- Where are documents stored?
- Where are operations stored?
- How do we recover after a server crash?

## Follow-up 7 — Large documents

- What happens if a document is 100 MB?
- Should every client receive the complete document?

## Follow-up 8 — Conflict resolution

- OT or CRDT?
- Why?
- How do we handle concurrent inserts/deletes?

## Follow-up 9 — Reliability

- What if a WebSocket server dies?
- What if Redis goes down?
- What if an operation is duplicated?

## Follow-up 10 — Security

- Who can edit a document?
- How are document permissions enforced?
- How do we prevent unauthorized WebSocket messages?

---

# 3. Requirements Clarification

Before designing, clarify the scope.

## Functional Requirements

### Document management

Users can:

- Create a document
- Rename a document
- Open a document
- Delete/archive a document
- Share a document
- Change permissions
- View document history
- Restore an older version

### Collaboration

Multiple users can:

- Edit simultaneously
- Insert text
- Delete text
- Replace text
- Format text
- See other users' changes
- See other users' cursors
- See selections

### Presence

Users can see:

- Online users
- Recently active users
- User cursor position
- User selection

### Comments

Users can:

- Add comments
- Reply to comments
- Resolve comments
- Mention users

### Offline mode

The client should:

- Continue local editing
- Queue operations
- Reconnect
- Synchronize pending operations
- Resolve conflicts

### Notifications

Users can receive:

- Mention notifications
- Comment notifications
- Share notifications
- Document activity notifications

---

# 4. Non-Functional Requirements

## Latency

Target:

```text
Local edit:
< 50 ms perceived response

Remote edit:
< 200 ms typical propagation

Presence:
~ 1 second acceptable

Comment:
< 1 second typical
```

These are design targets, not absolute guarantees.

## Availability

Target:

```text
99.99% for collaboration APIs
```

## Durability

Once an operation is acknowledged as durably committed:

```text
It must survive server restart/failure.
```

## Scalability

Example target:

```text
100M registered users
10M DAU
1M concurrent WebSocket connections
10K–100K document edits/sec globally
```

The exact numbers should be clarified during the interview.

## Consistency

Different operations have different requirements.

| Data | Consistency |
|---|---|
| Document operations | Strong ordering per document |
| Presence | Eventual |
| Cursor | Eventual |
| Comments | Strong enough for user-visible correctness |
| Permissions | Strong |
| Analytics | Eventual |

---

# 5. Scope Decision

Do not attempt to build everything initially.

## Version 1

Support:

```text
User
  |
Document
  |
Concurrent text editing
  |
WebSocket
  |
Operation ordering
  |
Persistence
  |
Presence
```

Later:

```text
Comments
Offline editing
Version history
Attachments
Search
Notifications
CRDT
Multi-region
```

---

# 6. High-Level Architecture

```text
                         ┌─────────────────────┐
                         │      Clients        │
                         │ React / Mobile      │
                         └──────────┬──────────┘
                                    │
                         HTTPS / WebSocket
                                    │
                         ┌──────────▼──────────┐
                         │   Load Balancer      │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
             ┌──────▼───────┐               ┌──────▼───────┐
             │ API Gateway   │               │ WebSocket    │
             │ / REST APIs   │               │ Gateway      │
             └──────┬───────┘               └──────┬───────┘
                    │                               │
          ┌─────────▼─────────┐          ┌─────────▼─────────┐
          │ Collaboration API │          │ Connection Manager │
          └─────────┬─────────┘          └─────────┬─────────┘
                    │                               │
                    └──────────────┬────────────────┘
                                   │
                         ┌─────────▼─────────┐
                         │ Collaboration     │
                         │ / Operation Engine │
                         └───────┬─────┬─────┘
                                 │     │
                 ┌───────────────┘     └────────────────┐
                 │                                      │
        ┌────────▼────────┐                    ┌────────▼────────┐
        │ Redis / PubSub  │                    │ Event Stream    │
        │ Presence        │                    │ Kafka           │
        └─────────────────┘                    └────────┬────────┘
                                                         │
                                  ┌──────────────────────┼─────────────────┐
                                  │                      │                 │
                           ┌──────▼──────┐       ┌──────▼──────┐   ┌──────▼──────┐
                           │ Document DB │       │ Snapshot    │   │ Audit/Event │
                           │             │       │ Storage     │   │ Storage     │
                           └─────────────┘       └─────────────┘   └─────────────┘
```

---

# 7. Major Components

## 7.1 Client

Responsible for:

- Local document model
- Rendering
- Local operation generation
- Optimistic UI
- Operation queue
- Conflict transformation/merging
- WebSocket connection
- Reconnection

Example:

```text
React UI
   |
Editor Model
   |
Operation Generator
   |
Local Operation Queue
   |
WebSocket Client
```

---

# 8. API Gateway

Responsibilities:

- Authentication
- Rate limiting
- Request routing
- TLS termination
- API versioning
- Request validation

Example:

```text
POST /api/v1/documents

GET /api/v1/documents/{documentId}

POST /api/v1/documents/{documentId}/share

GET /api/v1/documents/{documentId}/history
```

WebSocket:

```text
wss://api.example.com/collaboration
```

---

# 9. WebSocket Gateway

The collaboration system should not poll the server.

Instead:

```text
Client
   |
WebSocket
   |
WebSocket Gateway
```

A persistent connection allows:

```text
Client -> Server
Server -> Client
```

without repeated HTTP requests.

## Connection lifecycle

```text
CONNECT
  |
AUTHENTICATE
  |
JOIN_DOCUMENT
  |
SYNC
  |
READY
  |
EDIT <----------------------> EDIT
  |
LEAVE
  |
DISCONNECT
```

---

# 10. Why WebSocket?

Polling:

```text
Client -> GET
Client -> GET
Client -> GET
Client -> GET
```

creates unnecessary traffic.

WebSocket:

```text
Client <=================> Server
```

is better for:

- Low-latency edits
- Presence
- Cursor movement
- Notifications
- Server push

---

# 11. Core Problem: Concurrent Editing

Suppose the document is:

```text
HELLO
```

User A:

```text
Insert "X" at position 1
```

User B:

```text
Insert "Y" at position 1
```

Both may generate:

```text
Insert(X, 1)
Insert(Y, 1)
```

If different clients apply these in different orders:

```text
HXYELLO
```

vs

```text
HYXELLO
```

the documents diverge.

The collaboration engine must guarantee convergence.

---

# 12. Operation Model

Instead of sending the entire document:

```text
"Hello World..."
```

send operations.

Example:

```json
{
  "operationId": "op-123",
  "documentId": "doc-1",
  "clientId": "client-9",
  "baseVersion": 42,
  "type": "INSERT",
  "position": 5,
  "text": "abc"
}
```

Delete:

```json
{
  "operationId": "op-124",
  "documentId": "doc-1",
  "clientId": "client-9",
  "baseVersion": 43,
  "type": "DELETE",
  "position": 10,
  "length": 3
}
```

---

# 13. Operation Types

Start with:

```java
enum OperationType {
    INSERT,
    DELETE,
    RETAIN,
    FORMAT
}
```

Possible richer model:

```text
InsertOperation
DeleteOperation
FormatOperation
MoveOperation
ReplaceOperation
```

---

# 14. Versioning

Every document maintains a monotonically increasing version.

Example:

```text
Document version = 100

Operation A -> version 101
Operation B -> version 102
Operation C -> version 103
```

Client A knows:

```text
version = 100
```

and sends:

```text
operation(baseVersion=100)
```

Server currently has:

```text
version = 103
```

Therefore the operation cannot simply be applied blindly.

It must be transformed/merged against operations 101–103.

---

# 15. Option 1 — Operational Transformation

OT transforms concurrent operations so that they can be applied consistently.

Example:

Initial:

```text
ABC
```

A:

```text
Insert(X, 1)
```

B:

```text
Insert(Y, 1)
```

If A wins ordering:

```text
AXYBC
```

B's operation is transformed from:

```text
Insert(Y, 1)
```

to:

```text
Insert(Y, 2)
```

Therefore B becomes:

```text
AXYBC
```

Both clients converge.

---

# 16. OT Transformation Rules

For insert vs insert:

```text
I(p1, x)
I(p2, y)
```

If:

```text
p1 < p2
```

then second operation may remain unchanged.

If:

```text
p1 == p2
```

use deterministic tie-breaking:

```text
clientId
operationId
timestamp
```

Do not rely only on timestamps because distributed clocks are not perfectly synchronized.

---

# 17. Delete/Delete

Suppose:

```text
ABCDE
```

A deletes:

```text
B
```

B deletes:

```text
C
```

Original:

```text
Delete(position=1)
Delete(position=2)
```

After A deletes B:

```text
ACDE
```

B's delete position must shift:

```text
Delete(position=1)
```

Result:

```text
ADE
```

The transformation engine must handle overlapping deletes carefully.

---

# 18. OT Engine

Conceptually:

```java
interface OperationTransformer {

    TransformResult transform(
        Operation local,
        Operation remote
    );
}
```

Implementations:

```java
InsertInsertTransformer
InsertDeleteTransformer
DeleteInsertTransformer
DeleteDeleteTransformer
FormatTransformer
```

This follows the Open/Closed Principle.

New operation types can be introduced without rewriting the entire engine.

---

# 19. Alternative — CRDT

CRDT means:

> Conflict-free Replicated Data Type.

Instead of centrally transforming operations, the data structure is designed so replicas can merge operations deterministically.

Conceptually:

```text
Client A
   |
Replica A

Client B
   |
Replica B

Client C
   |
Replica C
```

Each replica can independently accept operations.

Eventually:

```text
Replica A = Replica B = Replica C
```

for the same set of operations.

---

# 20. OT vs CRDT

| Area | OT | CRDT |
|---|---|---|
| Central server | Natural | Not mandatory |
| Offline | Good | Excellent |
| Implementation | Complex | Complex |
| Memory overhead | Usually lower | Can be higher |
| Arbitrary distributed editing | Good | Very good |
| Proven document model | Mature | Mature |
| Server coordination | Higher | Lower |

For an interview design, it is reasonable to start with:

```text
Centralized OT
```

and discuss:

```text
CRDT as a future/alternative architecture
```

---

# 21. Collaboration Session

A document can have a collaboration session:

```text
DocumentSession
    |
    +-- Document ID
    +-- Current version
    +-- Active users
    +-- Connected clients
    +-- Operation manager
```

Example:

```java
class DocumentSession {

    private final DocumentId documentId;
    private long version;

    private final Map<ClientId, ClientConnection> clients;

    public void join(ClientConnection client) {}
    public void leave(ClientId clientId) {}
    public void broadcast(Operation operation) {}
}
```

---

# 22. Critical Scaling Question

> What happens if 10,000 users open the same document?

Do not create one isolated in-memory document copy per server.

Instead:

```text
                   Document 123
                       |
              ┌────────┴────────┐
              │                 │
        Collaboration       Event Stream
           Leader
              |
        Ordered operations
              |
      ┌───────┼────────┐
      │       │        │
   Server A Server B Server C
```

A logical document should have a single ordered operation stream.

---

# 23. Document Affinity

A practical design is:

```text
documentId -> collaboration partition
```

For example:

```text
hash(documentId) % N
```

maps a document to a logical partition.

Kafka can provide a similar partitioning model:

```text
Kafka partition key = documentId
```

Then operations for the same document are processed in order within the partition.

---

# 24. Kafka/Event Streaming

Use an event stream for durable asynchronous processing.

Example event:

```json
{
  "eventType": "DOCUMENT_OPERATION",
  "documentId": "doc-123",
  "operationId": "op-999",
  "version": 5001,
  "operation": {
    "type": "INSERT",
    "position": 20,
    "text": "Hello"
  }
}
```

Consumers can include:

```text
Persistence
Audit
Analytics
Notification
Search Indexer
Snapshot Service
```

---

# 25. Important Kafka Ordering Rule

Kafka guarantees ordering within a partition.

Therefore:

```text
key = documentId
```

is critical.

Bad:

```text
random partition
```

Good:

```text
partition = hash(documentId)
```

Then:

```text
doc-1 -> partition 2
doc-1 -> partition 2
doc-1 -> partition 2
```

Operations maintain partition ordering.

---

# 26. Redis

Redis can be used for:

### Presence

```text
document:123:presence
```

Example:

```text
user1 -> ONLINE
user2 -> ONLINE
user3 -> IDLE
```

### Connection metadata

```text
userId -> websocketServerId
```

### Pub/Sub

Useful for distributing ephemeral events.

However:

> Redis Pub/Sub should not be treated as the durable source of truth for document operations.

Use durable storage/event streaming for recoverability.

---

# 27. Document Storage

Possible design:

```text
PostgreSQL
```

for:

- Users
- Documents
- Permissions
- Comments
- Metadata
- Versions

Object storage:

```text
S3/Object Storage
```

for:

- Large snapshots
- Attachments
- Images
- Exported documents

Kafka:

```text
Operations/Event log
```

---

# 28. Snapshot Strategy

If a document has:

```text
10 million operations
```

replaying everything on every open is expensive.

Create snapshots.

Example:

```text
Snapshot at version 0

Operations:
1
2
3
...
1000

Snapshot at version 1000

Operations:
1001
1002
...
2000

Snapshot at version 2000
```

To recover:

```text
Load snapshot 2000
+
Replay operations 2001–2050
```

instead of:

```text
Replay 1–2050
```

---

# 29. Snapshot Frequency

Possible policy:

```text
Every 1000 operations
OR
Every 5 minutes
OR
when snapshot size/time threshold is reached
```

Make this configurable.

```java
interface SnapshotPolicy {

    boolean shouldSnapshot(
        long operationCount,
        Duration elapsed
    );
}
```

Implement:

```text
OperationCountSnapshotPolicy
TimeBasedSnapshotPolicy
HybridSnapshotPolicy
```

---

# 30. Data Model

## User

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP NOT NULL
);
```

## Document

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    title VARCHAR(500),
    current_version BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

## Document Permission

```sql
CREATE TABLE document_permissions (
    document_id UUID NOT NULL,
    user_id UUID NOT NULL,
    role VARCHAR(30) NOT NULL,
    PRIMARY KEY(document_id, user_id)
);
```

Roles:

```text
OWNER
EDITOR
COMMENTER
VIEWER
```

---

# 31. Operation Storage

```sql
CREATE TABLE document_operations (
    document_id UUID NOT NULL,
    version BIGINT NOT NULL,
    operation_id UUID NOT NULL,
    client_id VARCHAR(100) NOT NULL,
    operation_type VARCHAR(30) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,

    PRIMARY KEY(document_id, version),
    UNIQUE(operation_id)
);
```

Important:

```text
(document_id, version)
```

is the ordered sequence.

---

# 32. Idempotency

Networks can duplicate messages.

Client sends:

```text
operationId = abc
```

Server commits.

Client does not receive ACK.

Client retries:

```text
operationId = abc
```

Server must not apply it twice.

Use:

```text
UNIQUE(operation_id)
```

and/or an idempotency store.

---

# 33. WebSocket Protocol

Example client message:

```json
{
  "type": "OPERATION",
  "documentId": "doc-1",
  "operationId": "op-100",
  "baseVersion": 42,
  "payload": {
    "type": "INSERT",
    "position": 10,
    "text": "hello"
  }
}
```

Server ACK:

```json
{
  "type": "ACK",
  "operationId": "op-100",
  "version": 43
}
```

Broadcast:

```json
{
  "type": "REMOTE_OPERATION",
  "documentId": "doc-1",
  "version": 43,
  "operation": {}
}
```

---

# 34. Sync Protocol

When a client reconnects:

```text
Client:
"I have version 100"

Server:
"Current version = 108"

Server sends:
101
102
103
104
105
106
107
108
```

If the gap is huge:

```text
Client version = 100
Server version = 1,000,000
```

send:

```text
Snapshot(version=999500)
+
operations 999501–1000000
```

---

# 35. Reconnection

Client state:

```text
CONNECTED
    |
NETWORK_FAILURE
    |
DISCONNECTED
    |
RECONNECTING
    |
AUTHENTICATE
    |
SYNCING
    |
CONNECTED
```

Use exponential backoff:

```text
1 sec
2 sec
4 sec
8 sec
16 sec
30 sec
30 sec...
```

Add jitter to prevent reconnect storms.

---

# 36. Client-Side Operation Queue

Suppose the client loses connectivity.

User performs:

```text
A
B
C
D
```

Store locally:

```text
Pending:
A
B
C
D
```

After reconnect:

```text
Sync server state
      |
Transform/merge pending operations
      |
Send A
Send B
Send C
Send D
```

The client should not lose user work because of temporary network failure.

---

# 37. Presence System

Presence should be treated differently from document operations.

Example:

```text
WebSocket Gateway
      |
      v
Redis
```

Heartbeat:

```text
PING every 10 seconds
```

TTL:

```text
30 seconds
```

If heartbeat disappears:

```text
User considered offline
```

Presence does not need the same durability guarantees as document edits.

---

# 38. Cursor Updates

Cursor movement can generate huge traffic.

Do not persist every cursor movement.

Example:

```json
{
  "type": "CURSOR",
  "documentId": "doc-1",
  "userId": "user-1",
  "position": 245
}
```

Use:

```text
throttling
debouncing
coalescing
```

For example:

```text
send cursor update every 50–100 ms
```

rather than every mouse/keyboard event.

---

# 39. Comments Architecture

Comments should not be part of the core text operation stream unless the document model requires it.

Example:

```text
Comment
-------
id
documentId
authorId
anchor
text
createdAt
resolved
```

Anchor could use a stable document position/identifier rather than only a raw character offset.

Why?

Because character offsets change as users edit the document.

---

# 40. Class-Level Design

## Core interfaces

```java
public interface CollaborationService {

    JoinResult join(
        DocumentId documentId,
        UserId userId
    );

    void leave(
        DocumentId documentId,
        UserId userId
    );

    OperationResult apply(
        Operation operation
    );
}
```

---

# 41. Operation Interface

```java
public interface Operation {

    OperationId id();

    DocumentId documentId();

    long baseVersion();

    OperationType type();

    void apply(DocumentModel document);

    Operation inverse();
}
```

Implementations:

```java
public final class InsertOperation
        implements Operation {

    private final OperationId id;
    private final DocumentId documentId;
    private final long baseVersion;
    private final int position;
    private final String text;

    @Override
    public void apply(DocumentModel document) {
        document.insert(position, text);
    }

    @Override
    public Operation inverse() {
        return new DeleteOperation(...);
    }
}
```

---

# 42. Transformer Interface

```java
public interface OperationTransformer {

    Transformation transform(
        Operation local,
        Operation remote
    );
}
```

Implementation:

```java
public final class OtOperationTransformer
        implements OperationTransformer {

    @Override
    public Transformation transform(
        Operation local,
        Operation remote
    ) {
        // transformation rules
        return ...;
    }
}
```

This isolates conflict resolution from transport and persistence.

---

# 43. Operation Processor

```java
public interface OperationProcessor {

    OperationResult process(
        Operation operation
    );
}
```

Implementation:

```java
@Service
public class DefaultOperationProcessor
        implements OperationProcessor {

    private final OperationRepository repository;
    private final OperationTransformer transformer;
    private final DocumentRepository documentRepository;
    private final EventPublisher eventPublisher;

    @Override
    public OperationResult process(Operation operation) {
        // validate
        // check idempotency
        // load current version
        // transform if required
        // assign next version
        // persist
        // publish event
        // return ACK
    }
}
```

---

# 44. Repository Interfaces

```java
public interface DocumentRepository {

    Optional<Document> findById(DocumentId id);

    Document save(Document document);
}
```

```java
public interface OperationRepository {

    Optional<Operation> findById(OperationId id);

    List<Operation> findAfter(
        DocumentId documentId,
        long version
    );

    Operation save(Operation operation);
}
```

This follows Dependency Inversion.

The domain layer does not depend directly on PostgreSQL.

---

# 45. Event Publisher

```java
public interface CollaborationEventPublisher {

    void publish(OperationCommittedEvent event);
}
```

Kafka implementation:

```java
@Component
public class KafkaCollaborationEventPublisher
        implements CollaborationEventPublisher {
}
```

Future:

```text
RabbitMQ
Pulsar
NATS
```

can be introduced without modifying domain logic.

---

# 46. WebSocket Abstraction

```java
public interface ConnectionManager {

    void register(ClientConnection connection);

    void unregister(ClientId clientId);

    void send(
        ClientId clientId,
        ServerMessage message
    );

    void broadcast(
        DocumentId documentId,
        ServerMessage message
    );
}
```

Implementation:

```text
WebSocketConnectionManager
```

---

# 47. Session Manager

```java
public interface CollaborationSessionManager {

    DocumentSession getOrCreate(
        DocumentId documentId
    );

    void removeIfInactive(
        DocumentId documentId
    );
}
```

A session can maintain hot state:

```text
current version
active connections
recent operations
```

but must not become the only durable source of truth.

---

# 48. Complete Class Diagram

```text
                         ┌──────────────────────┐
                         │ CollaborationService │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ OperationProcessor   │
                         └──────┬──────┬────────┘
                                │      │
                 ┌──────────────┘      └──────────────┐
                 ▼                                     ▼
       ┌────────────────────┐              ┌─────────────────────┐
       │ OperationRepository│              │ OperationTransformer│
       └────────────────────┘              └─────────────────────┘
                 │                                     │
                 ▼                                     ▼
          ┌──────────────┐                    ┌────────────────┐
          │ PostgreSQL   │                    │ OT / CRDT      │
          └──────────────┘                    │ implementation  │
                                              └────────────────┘

       ┌─────────────────────┐
       │ DocumentRepository  │
       └──────────┬──────────┘
                  ▼
            ┌────────────┐
            │ PostgreSQL │
            └────────────┘

       ┌──────────────────────────┐
       │ CollaborationEventPublisher│
       └─────────────┬────────────┘
                     ▼
                  ┌───────┐
                  │ Kafka │
                  └───┬───┘
                      │
          ┌───────────┼───────────────┐
          ▼           ▼               ▼
     Persistence   Snapshot       Notification
       Consumer     Service          Service
```

---

# 49. SOLID Design

## Single Responsibility

Bad:

```text
CollaborationService
  - WebSocket
  - database
  - transformation
  - authentication
  - notification
```

Good:

```text
AuthenticationService
OperationProcessor
OperationTransformer
DocumentRepository
ConnectionManager
PresenceService
SnapshotService
```

Each has one major responsibility.

---

# 50. Open/Closed Principle

Bad:

```java
if (type == INSERT) ...
else if (type == DELETE) ...
else if (type == FORMAT) ...
```

everywhere.

Better:

```java
interface OperationHandler {

    boolean supports(OperationType type);

    void apply(Operation operation);
}
```

Implement:

```text
InsertHandler
DeleteHandler
FormatHandler
```

New operations can be added with minimal changes.

---

# 51. Liskov Substitution

Every implementation of:

```java
Operation
```

must obey the contract.

For example:

```text
InsertOperation
DeleteOperation
FormatOperation
```

must all be valid operations wherever an Operation is expected.

---

# 52. Interface Segregation

Avoid:

```java
interface CollaborationManager {

    join();
    leave();
    save();
    notify();
    comment();
    export();
    search();
}
```

Instead:

```text
SessionManager
OperationProcessor
CommentService
NotificationService
DocumentExporter
DocumentSearchService
```

---

# 53. Dependency Inversion

Business logic depends on:

```text
OperationRepository
EventPublisher
SnapshotRepository
```

not:

```text
PostgresOperationRepository
KafkaPublisher
S3SnapshotRepository
```

This allows infrastructure to change independently.

---

# 54. Request Flow — Edit

```text
1. User types "A"

2. Client creates operation

3. Client applies operation locally

4. Client sends WebSocket message

5. WebSocket Gateway authenticates

6. Collaboration Service validates permission

7. Operation Processor checks operationId

8. Processor loads current document version

9. If baseVersion is old:
       transform operation

10. Assign next version

11. Persist operation

12. Publish event

13. Broadcast operation

14. Client receives ACK

15. Other clients apply operation
```

---

# 55. Request Flow — Concurrent Edit

```text
Initial version = 10

Client A:
baseVersion = 10
Insert(X, 5)

Client B:
baseVersion = 10
Delete(5, 2)
```

Server receives A:

```text
version = 11
```

Server receives B:

```text
baseVersion = 10
current = 11
```

Therefore:

```text
B must be transformed against A
```

Then:

```text
B' = transform(B, A)
```

Apply:

```text
B'
```

and assign:

```text
version = 12
```

---

# 56. Failure Scenario — Server Crash

Suppose:

```text
WebSocket Server A
```

crashes.

Client reconnects to:

```text
WebSocket Server B
```

Client says:

```text
lastAcknowledgedVersion = 500
```

Server checks:

```text
currentVersion = 505
```

and synchronizes:

```text
501
502
503
504
505
```

If operations are persisted before ACK, no acknowledged edit is lost.

---

# 57. Failure Scenario — Database Failure

Do not acknowledge an operation as durable before the persistence contract is satisfied.

Possible flow:

```text
Receive operation
       |
Validate
       |
Persist
       |
Commit
       |
Publish
       |
ACK
```

For stronger event consistency, use:

```text
Transactional Outbox
```

---

# 58. Transactional Outbox

Problem:

```text
DB commit succeeds
Kafka publish fails
```

Now the database contains the operation but Kafka does not.

Solution:

```text
Database transaction
    |
    +-- document operation
    |
    +-- outbox event
```

Then:

```text
Outbox Publisher
       |
       ▼
Kafka
```

This avoids a database/Kafka dual-write inconsistency.

---

# 59. Exactly Once vs At Least Once

Do not assume network systems magically provide exactly-once behavior.

Prefer:

```text
At-least-once delivery
+
idempotent processing
```

Use:

```text
operationId
```

as the idempotency key.

---

# 60. Hot Document Problem

One document may become extremely popular.

Example:

```text
Document A
100,000 active users
```

A single partition could become overloaded.

Possible solutions:

### Solution 1

Limit active collaborators per document.

### Solution 2

Separate:

```text
editing stream
presence stream
cursor stream
```

### Solution 3

Use hierarchical fan-out:

```text
Operation Leader
      |
      +---- Fanout Node A
      +---- Fanout Node B
      +---- Fanout Node C
```

### Solution 4

Use specialized CRDT architecture for extreme distributed collaboration.

---

# 61. Fan-Out Strategy

Naive:

```text
Server receives operation
    |
for every user:
    send()
```

For:

```text
100,000 users
```

this is expensive.

Instead:

```text
Operation Stream
       |
Fanout Layer
       |
Connection Nodes
```

Each connection node handles only its connected clients.

---

# 62. Horizontal Scaling

WebSocket servers are stateful at the connection level.

Example:

```text
              Load Balancer
             /      |       \
            /       |        \
          WS1      WS2       WS3
```

Client A:

```text
doc-1 -> WS1
```

Client B:

```text
doc-1 -> WS2
```

Both must receive document operations.

Use:

```text
Kafka
Redis Pub/Sub
or another internal event bus
```

to distribute events.

---

# 63. Load Balancing

For WebSockets:

```text
Client
   |
Load Balancer
   |
WebSocket Server
```

Use:

```text
connection-aware load balancing
```

Sticky sessions can simplify connection management, but they should not be required for correctness.

If a connection moves to another node:

```text
client reconnects
syncs from last acknowledged version
```

---

# 64. Multi-Region Design

Global architecture:

```text
             Global DNS
                 |
       ┌─────────┴─────────┐
       │                   │
   Region A             Region B
       │                   │
 Collaboration        Collaboration
 Cluster A             Cluster B
       │                   │
     Kafka              Kafka
       │                   │
       └─────────┬─────────┘
                 │
          Global Storage
```

---

# 65. Multi-Region Document Ownership

Simpler model:

```text
documentId -> home region
```

Example:

```text
doc-1 -> Mumbai
doc-2 -> Singapore
doc-3 -> Frankfurt
```

Users can connect from anywhere, but operations are routed to the document's authoritative region.

This simplifies ordering.

---

# 66. Why Not Active-Active Immediately?

If:

```text
Region A
and
Region B
```

both independently order edits for the same document, conflict resolution becomes much harder.

Start with:

```text
single authoritative writer per document
```

and evolve toward multi-writer CRDT if business requirements justify it.

---

# 67. Security

Authentication:

```text
JWT / OAuth2 / session token
```

Authorization:

```text
documentId
    |
PermissionService
    |
OWNER / EDITOR / COMMENTER / VIEWER
```

Every WebSocket operation must be authorized.

Never assume:

```text
"If user joined the socket, they can edit."
```

---

# 68. WebSocket Security

At connection:

```text
Authenticate user
```

At document join:

```text
Check document permission
```

At operation:

```text
Validate:
- user
- document
- operation type
- permission
- payload size
- sequence/baseVersion
```

Rate-limit malicious clients.

---

# 69. Abuse Protection

Protect against:

```text
Huge document payload
Operation flooding
Connection flooding
Cursor flooding
Reconnect storms
Invalid operation spam
```

Use:

```text
per-user rate limits
per-document rate limits
message size limits
connection limits
circuit breakers
```

---

# 70. Observability

Metrics:

```text
websocket_connections
active_documents
operations_per_second
operation_latency
broadcast_latency
transform_latency
snapshot_latency
sync_duration
reconnect_rate
duplicate_operations
conflict_rate
```

Logs:

```text
operationId
documentId
clientId
userId
version
serverId
traceId
```

Do not log document contents by default because documents may contain sensitive data.

---

# 71. Distributed Tracing

Example:

```text
Client
 |
Gateway
 |
Collaboration Service
 |
Operation Processor
 |
Postgres
 |
Kafka
 |
Fanout
 |
Client
```

Propagate:

```text
traceId
```

through the flow.

---

# 72. Backpressure

Suppose clients produce:

```text
50K operations/sec
```

but processing capacity is:

```text
20K/sec
```

The system must avoid unlimited memory growth.

Use:

```text
bounded queues
```

and:

```text
backpressure
rate limiting
batching
load shedding
```

For cursor updates, coalesce intermediate updates.

For document operations, never silently drop committed edits.

---

# 73. Large Document Optimization

Never send:

```text
Entire document
```

on every edit.

Use:

```text
snapshot + incremental operations
```

Also consider:

```text
document chunks
lazy loading
binary protocol
compression
pagination
```

---

# 74. Binary Protocol

JSON is easy initially.

For high scale:

```text
Protobuf
MessagePack
custom binary format
```

can reduce:

```text
network bandwidth
serialization cost
GC pressure
```

However, do not prematurely optimize the protocol.

---

# 75. Database Scaling

Documents table:

```text
documents
```

can be partitioned/sharded by:

```text
documentId
```

Operations naturally partition by:

```text
documentId
```

Possible architecture:

```text
Shard 1 -> doc hash 0–999
Shard 2 -> doc hash 1000–1999
...
```

The exact sharding scheme should be selected according to traffic and operational requirements.

---

# 76. Read Path

Opening a document:

```text
GET /documents/{id}
```

Flow:

```text
Client
 |
API Gateway
 |
Document Service
 |
Cache?
 |
Snapshot Repository
 |
Operation Repository
 |
Document
```

Return:

```json
{
  "documentId": "doc-1",
  "version": 500,
  "content": "...",
  "permissions": {}
}
```

Then establish WebSocket synchronization.

---

# 77. Write Path

```text
WebSocket
   |
Authentication
   |
Authorization
   |
Validation
   |
Idempotency
   |
Transformation
   |
Persistence
   |
Outbox
   |
Kafka
   |
Fanout
   |
Clients
```

---

# 78. Cache Strategy

Cache:

```text
document metadata
permissions
recent snapshot
hot document state
```

Do not blindly cache mutable authoritative state without a clear invalidation/version strategy.

A versioned cache key helps:

```text
document:{id}:snapshot:{version}
```

---

# 79. CAP Discussion

For one document, we need:

```text
ordered operations
```

During a network partition, a strict single-writer design may temporarily sacrifice availability for that document rather than accept divergent authoritative histories.

CRDT-based architectures can make different trade-offs by allowing replicas to continue accepting operations and merging later.

The key interview point:

> CAP is about distributed-system trade-offs under partition; it is not a simple "choose two forever" switch.

---

# 80. Consistency Model

### Document edits

Strong ordering per document.

### Presence

Eventual consistency.

### Cursor

Eventual consistency.

### Analytics

Eventually consistent.

### Permissions

Strong consistency.

This is an example of:

> Different data paths can use different consistency models.

---

# 81. Version History

Store:

```text
snapshot
+
operation log
```

Users can view:

```text
Version 100
Version 200
Version 300
```

Restoration:

```text
version 300
    |
create new branch/revision
    |
continue editing
```

Avoid physically deleting newer history unless the product explicitly requires destructive rollback.

---

# 82. Audit Log

Record:

```text
who
what
when
document
operation
```

Example:

```json
{
  "actor": "user-123",
  "documentId": "doc-1",
  "action": "EDIT",
  "version": 500,
  "timestamp": "..."
}
```

Separate audit storage may be preferable for compliance.

---

# 83. Disaster Recovery

Back up:

```text
Document metadata
Operation log
Snapshots
Permissions
Comments
```

Recovery:

```text
Restore snapshot
+
Replay durable operations
```

Define:

```text
RPO
RTO
```

Example target:

```text
RPO < 1 minute
RTO < 15 minutes
```

Actual values depend on product requirements.

---

# 84. Testing Strategy

## Unit Tests

Test:

```text
Insert + Insert
Insert + Delete
Delete + Insert
Delete + Delete
```

and:

```text
same position
different position
overlapping ranges
empty operation
large operation
```

## Property Tests

Important properties:

```text
Convergence
Associativity where applicable
Idempotency
Operation validity
Version monotonicity
```

---

# 85. Convergence Test

Generate:

```text
Initial document
```

Create:

```text
Operation A
Operation B
Operation C
```

Apply in different valid orders.

Verify:

```text
Replica 1 == Replica 2 == Replica 3
```

This is one of the most important tests in a collaboration engine.

---

# 86. Load Testing

Simulate:

```text
100K WebSocket connections
```

and:

```text
10K edits/sec
```

Measure:

```text
p50
p95
p99
```

for:

```text
operation processing
broadcast
sync
reconnect
snapshot creation
```

---

# 87. Chaos Testing

Kill:

```text
WebSocket node
Kafka consumer
Redis node
database connection
network link
```

while users are editing.

Verify:

```text
No acknowledged operation disappears
Clients eventually reconnect
Documents converge
No duplicate operations are applied
```

---

# 88. Common Interview Mistakes

## Mistake 1

Sending the whole document for every keystroke.

### Better

Send operations/deltas.

---

## Mistake 2

Using Redis Pub/Sub as the source of truth.

### Better

Persist operations and use durable event streaming.

---

## Mistake 3

Ignoring duplicate messages.

### Better

Use operation IDs and idempotent processing.

---

## Mistake 4

Using timestamps to order concurrent edits.

### Better

Use server sequencing plus deterministic tie-breaking.

---

## Mistake 5

Persisting every cursor update.

### Better

Treat cursor/presence as ephemeral.

---

## Mistake 6

Assuming one WebSocket server can handle everything.

### Better

Horizontally scale connection nodes.

---

## Mistake 7

Ignoring offline clients.

### Better

Client-side pending operation queue + synchronization protocol.

---

# 89. Step-by-Step Interview Answer

When asked this question in an interview, answer in this sequence.

## Step 1 — Clarify requirements

Ask:

```text
Text-only or rich document?
Offline support?
Comments?
Presence?
Expected DAU?
Concurrent users/document?
Multi-region?
Strong consistency?
```

## Step 2 — Establish scale

Example:

```text
100M users
10M DAU
1M concurrent connections
100K edits/sec
```

## Step 3 — Define APIs

REST for:

```text
document management
permissions
comments
history
```

WebSocket for:

```text
edits
presence
cursor
real-time notifications
```

## Step 4 — Explain operation model

Introduce:

```text
INSERT
DELETE
FORMAT
```

## Step 5 — Solve conflicts

Explain:

```text
OT
```

then mention:

```text
CRDT alternative
```

## Step 6 — Explain ordering

Use:

```text
documentId -> partition
```

and:

```text
monotonic document version
```

## Step 7 — Explain persistence

Use:

```text
PostgreSQL
Kafka
Snapshots
Object storage
```

## Step 8 — Explain WebSocket scaling

Use:

```text
Load Balancer
WebSocket nodes
Redis/presence
Kafka/event fanout
```

## Step 9 — Explain failures

Cover:

```text
duplicate operation
server crash
database failure
network failure
reconnect
```

## Step 10 — Discuss future scaling

Cover:

```text
multi-region
CRDT
large documents
hot documents
binary protocol
```

---

# 90. Recommended Package Structure

For a Java/Spring implementation:

```text
collaboration/
├── domain/
│   ├── document/
│   │   ├── Document.java
│   │   ├── DocumentId.java
│   │   └── DocumentVersion.java
│   │
│   ├── operation/
│   │   ├── Operation.java
│   │   ├── InsertOperation.java
│   │   ├── DeleteOperation.java
│   │   └── FormatOperation.java
│   │
│   └── session/
│       └── DocumentSession.java
│
├── application/
│   ├── CollaborationService.java
│   ├── OperationProcessor.java
│   ├── PresenceService.java
│   ├── SnapshotService.java
│   └── CommentService.java
│
├── collaboration/
│   ├── OperationTransformer.java
│   ├── OtOperationTransformer.java
│   └── Transformation.java
│
├── infrastructure/
│   ├── postgres/
│   ├── kafka/
│   ├── redis/
│   ├── websocket/
│   └── objectstorage/
│
└── api/
    ├── rest/
    └── websocket/
```

---

# 91. SOLID Dependency Direction

```text
API
 |
Application
 |
Domain
 ^
 |
Infrastructure implements interfaces
```

The dependency should point inward.

For example:

```text
OperationProcessor
      |
      +--> OperationRepository
      +--> EventPublisher
      +--> OperationTransformer
```

not:

```text
OperationProcessor
      |
      +--> KafkaTemplate
      +--> JpaRepository
      +--> RedisTemplate
```

---

# 92. Future Enhancements

## Phase 1

```text
Basic collaborative text editing
WebSocket
OT
PostgreSQL
```

## Phase 2

```text
Presence
Cursor
Comments
Version history
Snapshots
```

## Phase 3

```text
Offline editing
Mobile clients
Attachments
Search
Notifications
```

## Phase 4

```text
CRDT
Multi-region
Global routing
Hot-document fanout
```

## Phase 5

```text
AI collaboration
Real-time AI suggestions
Document summarization
Smart conflict explanation
Semantic search
```

---

# 93. Advanced Follow-Up — AI Collaboration

An AI assistant could subscribe to:

```text
Document events
```

but should not block the critical edit path.

Bad:

```text
Edit
 |
AI processing
 |
Save
```

Better:

```text
Edit
 |
Persist
 |
Broadcast
 |
Kafka
 |
AI consumer
```

AI processing becomes asynchronous.

---

# 94. Advanced Follow-Up — Document Branching

Support:

```text
main
 |
 +-- branch-A
 |
 +-- branch-B
```

Useful for:

```text
code
design documents
legal documents
large reviews
```

The operation model can evolve into a version DAG rather than a simple linear history.

---

# 95. Advanced Follow-Up — Permissions

Permission changes must be handled carefully.

Example:

```text
User A is EDITOR
```

Admin changes:

```text
EDITOR -> VIEWER
```

The authorization service should reject subsequent edits.

Do not rely solely on an old WebSocket authorization decision.

---

# 96. Advanced Follow-Up — Search

Use asynchronous indexing:

```text
Document Operation
       |
       ▼
Kafka
       |
       ▼
Search Indexer
       |
       ▼
OpenSearch / Elasticsearch
```

Search should not run synchronously inside the edit transaction.

---

# 97. Advanced Follow-Up — Attachments

Large files should not travel through the collaboration WebSocket.

Use:

```text
Client
 |
Pre-signed upload URL
 |
Object Storage
```

Then store metadata:

```text
attachmentId
documentId
objectKey
size
mimeType
```

The document operation only references the attachment.

---

# 98. Final Architecture

```text
                              CLIENTS
                         ┌───────┴────────┐
                         │                │
                       REST           WebSocket
                         │                │
                         ▼                ▼
                  ┌────────────┐  ┌──────────────┐
                  │ API Gateway│  │ WS Gateway   │
                  └─────┬──────┘  └──────┬───────┘
                        │                 │
                        ▼                 ▼
                  ┌────────────┐  ┌──────────────┐
                  │ Document   │  │ Collaboration│
                  │ Service    │  │ Service      │
                  └─────┬──────┘  └──────┬───────┘
                        │                 │
                        │                 ▼
                        │          ┌──────────────┐
                        │          │ OT / CRDT    │
                        │          │ Engine        │
                        │          └──────┬───────┘
                        │                 │
                        │                 ▼
                        │          ┌──────────────┐
                        │          │ Operation    │
                        │          │ Sequencer    │
                        │          └──────┬───────┘
                        │                 │
                        │          ┌──────▼───────┐
                        │          │ PostgreSQL   │
                        │          │ Operation Log│
                        │          └──────┬───────┘
                        │                 │
                        │          ┌──────▼───────┐
                        │          │ Outbox       │
                        │          └──────┬───────┘
                        │                 │
                        │          ┌──────▼───────┐
                        │          │ Kafka        │
                        │          └──┬───┬───┬───┘
                        │             │   │   │
                        │             ▼   ▼   ▼
                        │          Snapshot Search
                        │          Audit   Notify
                        │
                        ▼
                  ┌──────────────┐
                  │ Snapshot /   │
                  │ Object Store │
                  └──────────────┘

                  ┌──────────────┐
                  │ Redis        │
                  │ Presence     │
                  │ Connection   │
                  └──────────────┘
```

---

# 99. Interview Summary

The core design can be remembered as:

```text
REAL-TIME COLLABORATION
        |
        +-- WebSocket
        |
        +-- Operation-based editing
        |
        +-- OT / CRDT
        |
        +-- Per-document ordering
        |
        +-- Idempotency
        |
        +-- Durable operation log
        |
        +-- Snapshots
        |
        +-- Kafka/event streaming
        |
        +-- Redis presence
        |
        +-- Horizontal WebSocket scaling
        |
        +-- Offline synchronization
        |
        +-- Permission enforcement
        |
        +-- Observability
```

The most important architectural principle is:

> **Separate the authoritative document operation path from ephemeral real-time state.**

Document edits require ordering, durability, idempotency, and convergence.

Presence and cursor updates prioritize low latency and can be lossy/eventually consistent.

---

# 100. Suggested Interview Follow-Up Sequence

Use these as an interview drill:

### Level 1 — Basics

1. Why WebSocket instead of HTTP polling?
2. What APIs would you expose?
3. What data would you store?

### Level 2 — Distributed systems

4. How do you handle concurrent edits?
5. How do you guarantee ordering?
6. What is OT?
7. What is CRDT?
8. OT vs CRDT?

### Level 3 — Reliability

9. What happens if the WebSocket server crashes?
10. What happens if an operation is delivered twice?
11. What happens if Kafka is unavailable?
12. How do you guarantee durability?

### Level 4 — Scale

13. How do you support 1M WebSocket connections?
14. What happens with 100K users editing one document?
15. How do you prevent a hot partition?
16. How would you shard the system?

### Level 5 — Advanced

17. How do you support offline editing?
18. How do you support multi-region collaboration?
19. How do you handle huge documents?
20. How do you design version history?
21. How would you introduce CRDT without rewriting the whole platform?

### Level 6 — Production

22. What metrics would you monitor?
23. How would you perform chaos testing?
24. How would you calculate capacity?
25. What are your RPO/RTO targets?
26. How would you protect document confidentiality?
27. How would you prevent malicious clients from flooding the collaboration channel?

---

# 101. Implementation Roadmap

If implementing this system from scratch, build it in this order:

```text
Step 1
Basic Document REST API

Step 2
Document persistence

Step 3
WebSocket connection

Step 4
Single-user operations

Step 5
Operation versioning

Step 6
Concurrent operation handling

Step 7
OT engine

Step 8
Operation persistence

Step 9
Idempotency

Step 10
Broadcast to multiple clients

Step 11
Redis presence

Step 12
Kafka event stream

Step 13
Transactional outbox

Step 14
Snapshots

Step 15
Reconnect + synchronization

Step 16
Offline operation queue

Step 17
Comments

Step 18
Version history

Step 19
Load testing

Step 20
Horizontal scaling

Step 21
Multi-region

Step 22
Evaluate CRDT
```

This order deliberately builds the **correctness model first**, then reliability, then scale and advanced distributed collaboration.

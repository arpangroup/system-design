# Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a project management and issue-tracking tool like JIRA. Cover both high-level design (services, data flow, scaling) and low-level design (class diagrams, core algorithms). Be ready to justify every design decision when I push back."*
>
> This guide is structured exactly as that interview unfolds: a requirements-gathering phase, a high-level architecture built up decision by decision, a low-level class design deep dive, and a sequence of escalating **follow-up questions** — each answered with real reasoning and, where it matters, real Java code — not a single static diagram presented as if no one ever questioned it.

---

# 1. What We Are Building

We are building **MiniTracker** — a project management and issue-tracking system covering:

- **Functional requirements**: projects, issues (bugs, stories, tasks, epics, sub-tasks), boards, sprints, customizable workflows, comments, attachments, notifications, and search/filtering.
- **High-level design**: service decomposition, API shape, data storage choices, real-time board updates, caching, multi-tenancy, and asynchronous processing.
- **Low-level design**: a full class model for the issue lifecycle, a genuinely pluggable workflow engine (the State pattern), role-based permissions, an event-driven notification system (the Observer pattern), and a JIRA-Query-Language-style filter evaluator.
- **Scaling concerns**: optimistic locking for concurrent edits, distributed unique ID generation for human-readable issue keys, sharding, and read/write separation.

```text
                        Client (web / mobile / CLI)
                                  |
                     API Gateway / Load Balancer
                                  |
        +---------+---------+---------+---------+---------+
        v         v         v         v         v         v
    Project   Issue    Workflow  Notification  Search   Auth/RBAC
    Service   Service   Service    Service     Service   Service
        |         |         |          |           |         |
        +---------+---------+----------+-----------+---------+
                                  |
                    Relational DB (source of truth)
                    + Search Index (Elasticsearch-style)
                    + Cache (Redis-style)
                    + Message Queue (async events)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Gather and state functional/non-functional requirements for an ambiguous system-design prompt, before designing anything.
- Justify a service decomposition (or a deliberate monolith) with concrete reasoning, not a reflexive "microservices are better."
- Design a class model where an issue's status transitions are validated by a genuinely pluggable **workflow engine**, not a hard-coded `if/else` chain.
- Design an event-driven side-effect system (notifications, webhooks, audit logs) using the Observer pattern, so adding a new side effect never touches existing code.
- Explain exactly how two users editing the same issue concurrently is resolved correctly, and how a system generates collision-free, human-readable IDs like `PROJ-123` at scale.
- Reason about what breaks first as data volume and concurrent users grow, and name the specific, standard techniques (sharding, read replicas, caching, rate limiting) that address each bottleneck.

---

# 3. Why This Matters (The Interview, Framed)

Designing a JIRA-like system is one of the most common **senior/staff application-architecture interview questions** because, unlike a narrower infrastructure question, it forces a candidate to move fluidly across **three different levels of design** in one conversation:

- **Requirements-driven scoping** — the prompt is deliberately broad ("design a project management tool"), and a strong candidate narrows it explicitly before designing, exactly as [the file storage guide's own opening](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) argues.
- **High-level architecture** — service boundaries, data flow, and the classic scaling toolkit (caching, queues, read replicas), tested through direct follow-up pressure rather than a memorized diagram.
- **Low-level, code-level design** — this is where many candidates who can draw boxes competently actually struggle: translating "issues have a status" into a real class model that supports a *customizable* workflow, real permissions, and real concurrent-edit handling, not just a `status: String` field.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and pattern implementations (State, Observer, Strategy). |
| Core data store | A relational database (Postgres-style, or [TinyDB](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) from the companion guide) | Issues, projects, and workflows have real relational structure and need real transactions (§52-§53). |
| Search index | An inverted-index-based search engine (Elasticsearch-style) | Full-text and faceted search over millions of issues doesn't fit a relational engine's strengths (§16-§17). |
| Cache | An in-memory key-value store (Redis-style) | Hot-path reads (a project's board, a user's assigned issues) benefit enormously from avoiding a database round trip (§20). |
| Message queue | A durable, ordered queue (Kafka-style) | Decouples "an issue changed" from "notify everyone who cares" (§23, §43-§46). |
| Real-time transport | WebSockets | Live board updates without polling (§18-§19). |

---

# 5. Project Structure

```text
minitracker/
├── src/main/java/com/example/minitracker/
│   ├── domain/
│   │   ├── Issue.java                    // §26
│   │   ├── IssueType.java, Priority.java, Label.java // §27
│   │   ├── Project.java                  // §28
│   │   ├── Epic.java, Story.java, SubTask.java // §29
│   │   ├── Board.java, Sprint.java        // §30-§31
│   │   └── User.java
│   ├── workflow/
│   │   ├── Workflow.java, WorkflowState.java, WorkflowTransition.java // §35
│   │   ├── TransitionGuard.java          // §37
│   │   └── IssueStatusContext.java       // §36
│   ├── permissions/
│   │   ├── Role.java, Permission.java     // §40
│   │   └── PermissionChecker.java        // §41
│   ├── events/
│   │   ├── IssueEvent.java, EventPublisher.java // §44-§45
│   │   ├── NotificationListener.java
│   │   ├── WebhookListener.java
│   │   └── AuditLogListener.java
│   ├── search/
│   │   ├── FilterExpression.java          // §50
│   │   └── FilterEvaluator.java           // §51
│   └── idgen/
│       └── IssueKeyGenerator.java         // §55
└── src/test/java/com/example/minitracker/
    ├── WorkflowTransitionTest.java
    ├── OptimisticLockingConcurrencyTest.java
    └── FilterEvaluatorTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"Is this single-tenant (one company's internal tool) or multi-tenant (a SaaS product serving many organizations)? Do we need fully customizable workflows per project, or is one fixed workflow acceptable? What's the expected scale — thousands or millions of issues? Do we need real-time collaborative updates, like seeing a teammate's comment appear live?"*

Exactly as [the file storage guide's opening](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) argues, jumping straight to boxes-and-arrows without narrowing "design a project management tool" is the first mistake a candidate can make — the answers to these questions determine whether multi-tenancy (§21-§22) and a pluggable workflow engine (§33-§38) are core requirements or premature complexity.

For this guide, we settle on a concrete, realistic scope: **multi-tenant SaaS**, **customizable per-project workflows**, targeting **millions of issues** and **tens of thousands of concurrent users**, with **real-time board updates**.

---

# 7. Functional Requirements

- Create and manage **projects**, each with its own team, workflow, and issue backlog.
- Create **issues** of multiple types (Bug, Story, Task, Epic, Sub-task) with a title, description, assignee, reporter, priority, labels, and status.
- Organize issues into **epics** (large bodies of work) and **sub-tasks** (smaller pieces of a single issue) — a real hierarchy, not a flat list.
- Group issues into **sprints** on a **board**, with drag-and-drop status transitions.
- Support **customizable workflows** per project (a project can define its own statuses and allowed transitions between them).
- Support **comments**, **attachments**, and an **activity log** on every issue.
- Support **role-based permissions** — who can create, edit, transition, or delete issues, per project.
- Support **search and filtering** across all issues, including a query-language-style filter (JIRA's JQL).
- Send **notifications** (in-app, email, webhook) when an issue relevant to a user changes.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Scalability** | Millions of issues, tens of thousands of concurrent users, without a redesign | Sharding (§57), read replicas (§58), caching (§20) |
| **Availability** | The tool stays usable even if a non-critical subsystem (search, notifications) is degraded | Asynchronous processing via a message queue (§23) decouples core writes from side effects |
| **Consistency** | An issue's core fields (status, assignee) must never silently lose a concurrent update | Optimistic locking (§52-§53) |
| **Low latency for reads** | Loading a board or an issue should feel instant | Caching (§20), a dedicated search index instead of ad-hoc relational queries (§16-§17) |
| **Extensibility** | Adding a new issue type, a new notification channel, or a new workflow status should not require touching unrelated code | The State (§36) and Observer (§44) patterns, applied deliberately for exactly this reason |

---

# 9. Follow-up Question 1 — "What's the Core Entity Model Before We Draw Any Boxes?"

> **Interviewer:** *"Before we talk about services or databases — what are the actual 'nouns' in this system? What has to exist, and how do they relate?"*

This question is a deliberate pivot away from architecture diagrams toward the **domain model** — and it's the correct order to design in: a service boundary or a database schema that doesn't reflect a clearly-understood domain model tends to need painful rework later. §10 answers it directly, and Part 3 (§25 onward) builds every one of these entities as real code.

---

# 10. Identifying the Core Domain Entities

| Entity | Represents | Key relationships |
|---|---|---|
| **Project** | A container for a team's work | Has many Issues, has one Workflow, has many Boards |
| **Issue** | A single unit of trackable work (bug, story, task) | Belongs to a Project, has a Status (via Workflow), may belong to an Epic, may have Sub-tasks |
| **Epic** | A large body of work spanning many issues | A special Issue type that "parents" other Issues |
| **Board** | A visual, often Kanban/Scrum-style view of issues | Belongs to a Project, filters/displays Issues, may be tied to a Sprint |
| **Sprint** | A fixed time-boxed iteration of work | Contains a subset of a Project's Issues |
| **Workflow** | The set of valid statuses and transitions for a Project's issues | Made of WorkflowStates and WorkflowTransitions (§35) |
| **User** | A person interacting with the system | Assigned to Issues, has Roles per Project (§40) |
| **Comment / Attachment** | Supplementary content on an Issue | Belongs to exactly one Issue |

Every section from §11 onward is either building infrastructure **around** these entities (services, storage, APIs) or building the entities **themselves** as real, working class designs (Part 3 onward) — nothing in this guide introduces a concept that isn't traceable back to this table.

---

# 11. High-Level Architecture Overview

At the highest level: clients talk to an API layer, which delegates to a set of focused services, each owning a slice of the domain model from §10, all ultimately persisting to a relational store, with a search index and cache as read-path accelerators and a message queue for anything that doesn't need to happen synchronously with the triggering request. §1's diagram is this section's answer, restated for reference; §12–§23 justify every box in it one decision at a time.

---

# 12. Follow-up Question 2 — "Monolith or Microservices? Justify It."

> **Interviewer:** *"You've drawn six separate services. Why not one application with six modules? What does splitting them into actual services buy you, concretely, and what does it cost?"*

The honest answer resists the reflexive "microservices are more scalable" — the real justification is about **independent scaling and failure isolation**, not scalability in the abstract: the **Search Service** (§16-§17) has a completely different resource profile (index-heavy, CPU for query parsing) than the **Notification Service** (§43-§46, I/O-bound, bursty), and a spike in one shouldn't be able to starve the other of resources if they're deployed and scaled independently. The cost, honestly stated, is real: network calls between services replace in-process method calls (added latency, added failure modes), and data consistency across service boundaries becomes eventual rather than transactional (§23's message queue exists specifically to manage this tradeoff). For a **smaller** version of this system (§6's scope questions matter here too), a well-modularized monolith is a legitimate, often better, answer — the services-vs-monolith decision should be justified by the specific scale and failure-isolation needs established in requirements, not asserted as a default.

---

# 13. Service Decomposition: Project, Issue, Workflow, Notification, Search

| Service | Owns | Talks to |
|---|---|---|
| **Project Service** | Project CRUD, team membership, board/sprint configuration | Issue Service (to list a project's issues) |
| **Issue Service** | Issue CRUD, the core domain model (§25-§32) | Workflow Service (validate transitions), Event Publisher (§45) |
| **Workflow Service** | Workflow definitions, valid states/transitions per project (§33-§38) | Issue Service (called synchronously on every status change) |
| **Notification Service** | Delivering in-app/email/webhook notifications | Consumes events from the message queue (§23), asynchronously |
| **Search Service** | Indexing issues, executing filter/search queries (§16-§17, §49-§51) | Consumes issue-changed events to keep its index current |
| **Auth/RBAC Service** | Authentication, role and permission checks (§39-§42) | Called by every other service on every write |

Each service owns exactly one clear slice of §10's domain model, and — critically — each one could, in principle, be scaled, deployed, and even rewritten independently without the others needing to change, which is the actual test of whether a service boundary is well-drawn.

---

# 14. API Design: The Core REST Endpoints

```text
POST   /projects                          create a project
GET    /projects/{projectId}/issues       list issues in a project (supports filter query params, §49-§51)
POST   /projects/{projectId}/issues       create an issue
GET    /issues/{issueKey}                 get a single issue by its human-readable key (e.g. PROJ-123)
PATCH  /issues/{issueKey}                 update fields (uses optimistic locking, §52-§53)
POST   /issues/{issueKey}/transitions     move an issue to a new status (validated by the Workflow Service, §33-§38)
POST   /issues/{issueKey}/comments        add a comment
GET    /search?jql=...                    execute a filter query (§49-§51)
```

Note that **transitioning** an issue's status is deliberately its **own** endpoint (`POST /transitions`), not folded into the general `PATCH` — this is a direct consequence of §33-§38's design: a transition isn't an ordinary field update, it's a validated operation against the workflow engine, and giving it its own endpoint makes that distinction visible at the API layer, not just buried inside a generic update handler's internal logic.

---

# 15. Data Storage Choices: Why a Relational Store for Core Data

Projects, issues, and their relationships (an issue belongs to a project, may belong to an epic, has a status from a specific workflow) are fundamentally **relational** — foreign keys, joins, and, crucially, **transactions** (creating an issue and appending its first activity-log entry must succeed or fail together) are exactly what a relational database is built for. [TinyDB's own transaction machinery](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) — atomicity, isolation, rollback — is precisely the guarantee the core Issue/Project data needs and a document or pure key-value store would need to reconstruct manually.

---

# 16. Follow-up Question 3 — "How Do You Handle Full-Text Search Across Millions of Issues?"

> **Interviewer:** *"A user wants to search for 'issues mentioning null pointer exception in the payment service, assigned to Alice, updated in the last week.' Walk me through how that query actually executes."*

Not via the relational database directly — a `LIKE '%null pointer%'` query has no efficient index support for free-text search at this scale, and combining free-text relevance with structured filters (assignee, date range) in one relational query is exactly the wrong tool for the job. The answer is a **dedicated search index** (§17), kept **eventually consistent** with the source-of-truth relational data via the same event stream (§23) the Notification Service already consumes.

---

# 17. Search Architecture: A Secondary Search Index (Elasticsearch-style)

```text
Issue Service: issue created/updated -> writes to relational DB (source of truth)
                                      -> publishes an IssueChanged event to the message queue (§23)
                                                    |
                                                    v
                                        Search Indexer (a consumer)
                                                    |
                                                    v
                                     Inverted index: term -> [issue ids]
                                     (tokenized title/description, plus structured fields
                                      like assignee/status/labels for fast filtering)
```

A query like §16's example decomposes into a free-text relevance search (`"null pointer exception"`, `"payment service"`) **combined with** structured filters (`assignee = Alice`, `updatedAt > now - 7d`) — exactly what an inverted-index search engine is purpose-built to execute efficiently, at a cost the design explicitly accepts: the search index can be **milliseconds to seconds behind** the source of truth, an eventual-consistency tradeoff that's entirely acceptable for a search/discovery use case (nobody expects a search result to reflect a write from half a second ago) but would be **unacceptable** for the core issue read/write path itself (§15).

---

# 18. Follow-up Question 4 — "How Do Real-Time Board Updates Work When Multiple Users Are Viewing the Same Board?"

> **Interviewer:** *"Two teammates have the same Scrum board open. One drags an issue to 'In Progress.' How does the other person's screen update without them refreshing?"*

Polling (repeatedly asking "anything new?" on a timer) works but wastes bandwidth and adds latency proportional to the poll interval. The real answer is a **persistent connection** (WebSockets) from each connected client to the server, paired with a **publish-subscribe** mechanism so a write from any client fans out to every other client currently watching the same board.

---

# 19. Real-Time Updates: WebSockets and a Pub/Sub Backbone

```text
User A drags an issue -> PATCH /issues/{key} (or POST /transitions)
                       -> Issue Service processes the write, publishes an "IssueChanged" event
                                                      |
                                                      v
                                    Pub/Sub backbone (topic: "board:{boardId}")
                                                      |
                              +----------------------+----------------------+
                              v                                             v
                    WebSocket connection                          WebSocket connection
                    (User A's browser)                             (User B's browser)
                              |                                             |
                    (already knows — it made the change)          receives the update, re-renders LIVE
```

The same event that feeds the search indexer (§17) and the notification service (§23, §45) also feeds this real-time fan-out — **one** event, **multiple** independent consumers, each reacting to it for a completely different purpose. This is the direct payoff of treating "an issue changed" as a first-class published event rather than a side effect buried inside the Issue Service's own write path.

---

# 20. Caching Strategy: What to Cache and Why

| What's cached | Why | Invalidation |
|---|---|---|
| A project's board view (issues + their current status/assignee) | Read far more often than written — a board is loaded on every page view but only mutated on individual drag-and-drop actions | Invalidate the specific issue's cache entry on the `IssueChanged` event (§17, §19) reused again here |
| A user's "assigned to me" issue list | Loaded on nearly every page (a persistent sidebar/notification badge) | Same event-driven invalidation |
| Workflow definitions per project | Rarely change, read on every single status transition validation (§36-§37) | A much longer TTL, or invalidated only on an explicit workflow edit |

The pattern worth naming explicitly: caching pays off most where **read frequency vastly exceeds write frequency**, and the same `IssueChanged` event this guide already introduced for search (§17) and real-time updates (§19) is, once again, the natural invalidation trigger — one event, a third independent consumer.

---

# 21. Follow-up Question 5 — "How Would You Design for Multi-Tenancy (Many Organizations, One Platform)?"

> **Interviewer:** *"This is a SaaS product — thousands of different companies use it, each with their own projects and users, who must never see each other's data. How do you architect that?"*

Three standard approaches exist, each a different point on a spectrum trading isolation strength against operational complexity — §22 lays them out directly rather than picking one without showing the tradeoff.

---

# 22. Multi-Tenancy: Shared Schema vs Schema-per-Tenant vs Database-per-Tenant

| Approach | How it works | Isolation | Operational cost |
|---|---|---|---|
| **Shared schema** | Every table has a `tenant_id` column; every query filters on it | Weakest — a missing `WHERE tenant_id = ?` is a real, dangerous bug class | Lowest — one schema, one set of migrations, cheapest to run at small-to-medium scale |
| **Schema-per-tenant** | Each tenant gets its own schema within a shared database instance | Stronger — accidental cross-tenant queries are structurally harder | Moderate — migrations must run per-schema, but infrastructure is still shared |
| **Database-per-tenant** | Each tenant gets a fully separate database (or even separate infrastructure) | Strongest — a bug literally cannot leak data across tenants at the query level | Highest — provisioning, migrating, and monitoring scale linearly with tenant count |

For MiniTracker's assumed scale (§6), **shared schema with a mandatory `tenant_id`, enforced at the ORM/query-builder level rather than trusted to every hand-written query**, is the pragmatic starting choice — with schema- or database-per-tenant reserved as an upgrade path for a small number of enterprise customers with contractual isolation requirements, a real, common pattern rather than an all-or-nothing choice.

---

# 23. Asynchronous Processing: A Message Queue for Notifications and Webhooks

Sending an email, calling a customer's webhook URL, and updating a search index all share a property that matters architecturally: **none of them need to complete before the triggering write is acknowledged to the user**. Forcing them to be synchronous would mean an issue update's latency is at the mercy of the slowest side effect (a webhook endpoint that takes 3 seconds to respond) — a durable, ordered **message queue** decouples "the write succeeded" from "everything that should eventually happen because of it," letting the Issue Service publish one event and move on immediately (§17, §19, §20 all already lean on this same event as consumers).

---

# 24. High-Level Architecture Diagram, Assembled

```text
                              Client (web / mobile)
                                       |
                          API Gateway (auth, rate limiting §59)
                                       |
        +----------+----------+----------+----------+----------+
        v          v          v          v          v          v
    Project    Issue      Workflow   Auth/RBAC   (reads hit cache first, §20)
    Service    Service    Service     Service
        |          |          |
        +----------+----------+
                   |
        Relational DB (source of truth, §15) --- tenant_id on every row (§22)
                   |
          publishes IssueChanged events
                   |
        +----------+----------+----------+
        v                     v          v
  Search Indexer      Notification    WebSocket Pub/Sub
  (§17)               Service (§23)   (§19, live board updates)
        |                     |
        v                     v
  Search Index          Email / Webhook delivery
  (Elasticsearch-style)  (async, via the queue)
```

Every box in this diagram was justified by a specific follow-up question (§12, §16, §18, §21) rather than assumed — which is exactly the difference between presenting an architecture and defending one.

---

# 25. Follow-up Question 6 — "Let's Get Concrete. Design the Class Model for an Issue and Its Lifecycle."

> **Interviewer:** *"We've covered architecture at a high level. Now let's go deep on one piece: design the actual classes for an Issue, including its type, its status, and how it relates to epics and sub-tasks. I want to see real fields and real relationships, not a box labeled 'Issue.'"*

This is the pivot every strong system-design interview makes eventually — from architecture to **class-level design**, and it's where a candidate's actual object-oriented design skill (not just their familiarity with architecture buzzwords) becomes visible. §26–§32 answer it in full.

---

# 26. The Issue Entity

```java
// domain/Issue.java
public class Issue {
    private final String key;              // e.g. "PROJ-123" — human-readable, generated per §54-§55
    private final String projectId;
    private IssueType type;                 // §27
    private String title;
    private String description;
    private Priority priority;              // §27
    private final Set<Label> labels = new HashSet<>(); // §27
    private String assigneeUserId;
    private final String reporterUserId;
    private WorkflowState status;           // §35 — NOT a plain enum; see §34 for why
    private String epicKey;                 // null unless this issue belongs to an Epic (§29)
    private final List<String> subTaskKeys = new ArrayList<>(); // §29
    private long version;                   // optimistic locking, §52-§53
    private final Instant createdAt;
    private Instant updatedAt;

    // constructor / getters / setters omitted for brevity — every SETTER that changes status
    // goes through WorkflowService.transition(), never a direct field assignment, per §36-§37
}
```

Two fields are deliberately **not** what a first-pass design might reach for: `status` is a `WorkflowState` object (§35), not a plain `enum Status { TODO, IN_PROGRESS, DONE }` — because §34 shows exactly why a fixed enum can't support §7's "customizable workflow per project" requirement. And `subTaskKeys`/`epicKey` model the hierarchy from §10 directly as references by key, not nested objects — an issue hierarchy that's too deep to load recursively by default, loaded on demand instead.

---

# 27. IssueType, Priority, Label — Value Objects and Enums

```java
public enum IssueType { BUG, STORY, TASK, EPIC, SUB_TASK }
public enum Priority { LOWEST, LOW, MEDIUM, HIGH, HIGHEST }

// Label is a per-project, user-defined tag — NOT a fixed enum, since projects need to invent their own labels
public record Label(String name, String colorHex) { }
```

`IssueType` and `Priority` are fixed enums because they're genuinely closed sets defined by the *system*, not by individual users — every project shares the same five priority levels. `Label`, by contrast, is user-defined per project ("frontend", "needs-design", "customer-reported") and modeled as a plain value object stored in the database, not a code-level enum — the same "closed set defined by the domain vs. open set defined by users" distinction that should drive every enum-vs-database-row decision in this kind of system.

---

# 28. The Project Entity and Its Relationship to Issues

```java
// domain/Project.java
public class Project {
    private final String id;
    private final String tenantId;          // §22 — every row in a multi-tenant table carries this
    private String key;                     // e.g. "PROJ" — the prefix used in every issue's human-readable key, §54-§55
    private String name;
    private Workflow workflow;               // §35 — THIS project's specific set of valid states/transitions
    private final List<String> boardIds = new ArrayList<>();

    // getIssues() is deliberately NOT a field here — issues are queried from the Issue Service/repository
    // by projectId, never held as an in-memory collection on Project itself (this would not scale to
    // a project with 100,000 issues, and it's not how any real system models a one-to-many at this size)
}
```

The comment on the missing `issues` field is worth taking seriously as a design decision, not an omission: a naive `List<Issue> issues` field on `Project` looks natural in a UML diagram but is a real anti-pattern once a project can have tens of thousands of issues — it invites loading the entire collection into memory for an operation that only needed one issue, or three.

---

# 29. Epics, Stories, and Sub-Tasks: Modeling Issue Hierarchy

Rather than three unrelated classes, `Epic`, `Story`, and `SubTask` are all just `Issue` objects with `type` set accordingly (§27) — the hierarchy lives entirely in the **relationship fields** already present on `Issue` (§26):

```text
Epic (Issue, type=EPIC)
  |
  +-- Story (Issue, type=STORY, epicKey = the Epic's key)
  |     |
  |     +-- SubTask (Issue, type=SUB_TASK, referenced in the Story's subTaskKeys)
  |
  +-- Bug (Issue, type=BUG, epicKey = the Epic's key)
```

This is a deliberate, important simplification over "one Java class per issue type": every issue type shares the *overwhelming majority* of its behavior (has a status, a workflow, comments, an activity log, participates in a board) — introducing a full class hierarchy (`Epic extends Issue`, `Story extends Issue`) would mean every piece of shared logic (querying, workflow validation, permission checks) needs to handle a polymorphic type, for essentially no behavioral benefit, since the *only* real difference between the types is which fields are meaningful and what parent/child relationships are valid — exactly the kind of case where composition (a `type` field plus relationship fields) beats inheritance.

---

# 30. The Board: Sprints, Backlogs, and Columns

```java
// domain/Board.java
public class Board {
    private final String id;
    private final String projectId;
    private BoardType type;                 // SCRUM or KANBAN
    private final List<BoardColumn> columns = new ArrayList<>(); // §35's WorkflowStates, mapped to visual columns
    private String activeSprintId;           // null for a Kanban board, or a Scrum board with no active sprint
}

public record BoardColumn(String name, List<WorkflowState> mappedStates) { }
```

A board's columns are explicitly a **mapping onto** the project's workflow states (§35), not an independent concept — a "To Do" column might map to a single `TODO` workflow state, while a "Done" column could map to *both* `DONE` and `WONT_FIX` states collapsed into one visual column. Modeling this as an explicit mapping, rather than assuming a 1:1 correspondence between workflow states and board columns, is what supports a real JIRA feature (multiple statuses shown in one column) without any special-casing elsewhere in the design.

---

# 31. The Sprint Entity and Its Lifecycle

```java
// domain/Sprint.java
public class Sprint {
    private final String id;
    private final String boardId;
    private String name;
    private SprintState state;               // PLANNED -> ACTIVE -> COMPLETED — itself a small, fixed state machine
    private Instant startDate, endDate;
    private final Set<String> issueKeys = new HashSet<>(); // which issues are committed to this sprint

    public void start() {
        if (state != SprintState.PLANNED) throw new IllegalStateException("Only a PLANNED sprint can be started");
        this.state = SprintState.ACTIVE;
    }
    public void complete(List<String> incompleteIssueKeysToCarryOver) {
        if (state != SprintState.ACTIVE) throw new IllegalStateException("Only an ACTIVE sprint can be completed");
        this.state = SprintState.COMPLETED;
        // incomplete issues are moved to the backlog or the next sprint — handled by the caller, given this list
    }
}
```

`Sprint`'s own `PLANNED -> ACTIVE -> COMPLETED` progression is a small state machine in its own right, guarded the same defensive way (checking current state before allowing a transition) that §36's much larger `WorkflowState` engine guards issue status changes — the same design idea, applied at a smaller scale, because it's genuinely useful any time an object has a small number of valid states and not every transition between them is legal.

---

# 32. Class Diagram: Core Domain Entities

```text
┌─────────────┐        1      *  ┌─────────────┐        *      1  ┌─────────────┐
│   Project   │─────────────────>│    Issue    │<─────────────────│    Board    │
├─────────────┤   has many        ├─────────────┤   filters/shows  ├─────────────┤
│ id          │                   │ key         │                   │ id          │
│ tenantId    │                   │ projectId   │                   │ projectId   │
│ key         │        1      1   │ type        │                   │ type        │
│ workflow ───┼──────────────────>│ status      │                   │ columns     │
└─────────────┘   defines valid   │ priority    │                   │ activeSprint│
                   states for     │ labels      │                   └──────┬──────┘
                                  │ assignee    │                          │ 1
                                  │ epicKey     │                          │
                                  │ subTaskKeys │                          v *
                                  │ version     │                   ┌─────────────┐
                                  └──────┬──────┘                   │   Sprint    │
                                         │ 1                        ├─────────────┤
                                         │                          │ id          │
                                         v *                        │ state       │
                                  ┌─────────────┐                   │ issueKeys   │
                                  │  Comment /  │                   └─────────────┘
                                  │  Attachment │
                                  └─────────────┘
```

Every box and every cardinality in this diagram traces back to a decision already justified in §26–§31 — a `1..*` between `Project` and `Issue`, for instance, is a direct consequence of §28's decision *not* to hold issues as an in-memory collection on `Project`.

---

# 33. Follow-up Question 7 — "How Do You Model a Customizable Workflow (To Do → In Progress → Done, or a Custom One)?"

> **Interviewer:** *"§7 said workflows are customizable per project. One team wants a simple three-status flow. Another wants five statuses with a mandatory code-review step before 'Done.' How does your class design support both without an if/else chain checking project IDs?"*

This is the question that separates "I used an enum for status" from a genuinely extensible design — §34 names exactly why a fixed enum fails this requirement, and §35–§37 build the actual engine that succeeds at it.

---

# 34. Why a Hard-Coded Status Enum Doesn't Scale

```java
// The tempting, WRONG first approach
public enum Status { TODO, IN_PROGRESS, IN_REVIEW, DONE }

public void transition(Issue issue, Status newStatus) {
    if (issue.getStatus() == Status.TODO && newStatus == Status.IN_PROGRESS) { /* ok */ }
    else if (issue.getStatus() == Status.IN_PROGRESS && newStatus == Status.IN_REVIEW) { /* ok */ }
    // ... one branch per valid transition, for EVERY project, hard-coded at compile time ...
    else throw new IllegalStateException("Invalid transition");
}
```

This compiles, works for a demo, and fails the actual requirement immediately: every project is forced into the **same** fixed set of statuses and the **same** fixed set of valid transitions between them — exactly what §7/§33 said must be customizable per project. Worse, adding a new status for *any* project means editing this shared enum and this shared `if/else` chain, risking every other project's workflow in the process. The fix isn't a bigger enum — it's recognizing that **states and transitions are data, not code** (§35).

---

# 35. The Workflow, State, and Transition Model

```java
// workflow/WorkflowState.java
public record WorkflowState(String id, String name, boolean isInitial, boolean isTerminal) { }

// workflow/WorkflowTransition.java
public record WorkflowTransition(String id, WorkflowState fromState, WorkflowState toState,
                                  String name, List<TransitionGuard> guards) { } // §37

// workflow/Workflow.java — this IS the per-project customization point §33 asked about
public class Workflow {
    private final String id;
    private final List<WorkflowState> states = new ArrayList<>();
    private final List<WorkflowTransition> transitions = new ArrayList<>();

    public List<WorkflowTransition> getAvailableTransitions(WorkflowState currentState) {
        return transitions.stream().filter(t -> t.fromState().equals(currentState)).toList();
    }

    public boolean isValidTransition(WorkflowState from, WorkflowState to) {
        return transitions.stream().anyMatch(t -> t.fromState().equals(from) && t.toState().equals(to));
    }
}
```

Every project gets its **own** `Workflow` instance, built from its own set of `WorkflowState`/`WorkflowTransition` rows stored in the database — a simple three-status project and a five-status project with a mandatory review gate are just two different **data** configurations of the exact same classes, with zero code branching on which project is which. This is the direct, concrete resolution of §34's problem: states and transitions moved from hard-coded Java control flow into data the `Workflow` class interprets generically.

---

# 36. Implementing the State Pattern for Issue Status Transitions

```java
// workflow/IssueStatusContext.java — the State pattern's "Context," coordinating a transition attempt
public class IssueStatusContext {
    private final Issue issue;
    private final Workflow workflow;

    public IssueStatusContext(Issue issue, Workflow workflow) {
        this.issue = issue;
        this.workflow = workflow;
    }

    public void transitionTo(WorkflowState targetState, TransitionContext ctx) {
        if (!workflow.isValidTransition(issue.getStatus(), targetState)) {
            throw new InvalidTransitionException(issue.getStatus(), targetState);
        }
        WorkflowTransition transition = findTransition(issue.getStatus(), targetState);
        for (TransitionGuard guard : transition.guards()) {         // §37
            guard.check(issue, ctx); // throws if this specific transition's precondition isn't met
        }
        issue.setStatus(targetState); // the ONLY place in the entire codebase allowed to mutate Issue.status
        eventPublisher.publish(new IssueTransitionedEvent(issue, transition)); // §44-§45
    }
    // findTransition(...) omitted for brevity — looks up the matching WorkflowTransition from workflow.getAvailableTransitions
}
```

This is a direct application of the **State pattern**: `Issue.status` never changes except through `IssueStatusContext.transitionTo`, which centralizes **every** rule about what a valid transition is (§35's data-driven check) and what must be true for it to be allowed (§37's guards) — no other code path in the system can move an issue into an invalid or unauthorized state, because no other code path is given the ability to.

---

# 37. Validating Transitions and Enforcing Guards (e.g., "Can't Close Without QA Approval")

```java
// workflow/TransitionGuard.java
public interface TransitionGuard {
    void check(Issue issue, TransitionContext ctx); // throws GuardFailedException if the precondition isn't met
}

public class RequiresQaApprovalGuard implements TransitionGuard {
    @Override
    public void check(Issue issue, TransitionContext ctx) {
        if (!ctx.hasLabel(issue, "qa-approved")) {
            throw new GuardFailedException("Cannot transition to Done without the 'qa-approved' label");
        }
    }
}

public class RequiresRoleGuard implements TransitionGuard {
    private final String requiredRole;
    @Override
    public void check(Issue issue, TransitionContext ctx) {
        if (!ctx.currentUserHasRole(requiredRole)) {
            throw new GuardFailedException("Only a " + requiredRole + " can perform this transition");
        }
    }
}
```

A `WorkflowTransition` (§35) holds a **list** of `TransitionGuard`s — this is the Strategy pattern layered on top of the State pattern: which specific guard(s) apply to a given transition is, again, **data**, configured per project per transition, not a hard-coded check. A project that wants "only a Team Lead can reopen a closed issue" attaches a `RequiresRoleGuard` to that one transition; a project that doesn't care attaches none — exactly the extensibility §7's requirement demanded, now genuinely delivered rather than promised.

---

# 38. Class Diagram: The Workflow Engine

```text
┌──────────────┐   1        *   ┌───────────────────┐
│   Workflow   │───────────────>│ WorkflowTransition │
├──────────────┤   has many      ├───────────────────┤
│ id           │                 │ fromState          │
│ states[]     │   1        *    │ toState             │
│ transitions[]│<───────────────>│ guards: List<Guard>│──────┐
└──────────────┘                 └───────────────────┘      │ *
       ^                                                     v
       │ references                                  ┌───────────────┐
       │                                              │TransitionGuard│ (interface)
┌──────────────┐                                      ├───────────────┤
│ WorkflowState│                                       │ + check(...)  │
├──────────────┤                                      └───────┬───────┘
│ id, name     │                                              │ implements
│ isInitial    │                                    ┌─────────┴─────────┐
│ isTerminal   │                                    v                   v
└──────────────┘                          ┌──────────────────┐ ┌──────────────────┐
       ^                                  │RequiresQaApproval│ │  RequiresRole    │
       │ current state of                 │      Guard        │ │      Guard       │
       │                                  └──────────────────┘ └──────────────────┘
┌──────────────┐
│    Issue     │   (from §26 — status field IS a WorkflowState)
└──────────────┘
```

This diagram is the concrete answer to §33's original challenge — nowhere in it does a project ID appear in any conditional logic; **every** project-specific behavior lives in which `WorkflowState`/`WorkflowTransition`/`TransitionGuard` **data** that project's `Workflow` instance happens to be composed of.

---

# 39. Follow-up Question 8 — "How Do You Handle Permissions — Who Can Do What, on Which Project?"

> **Interviewer:** *"Alice is an admin on Project A but just a regular contributor on Project B. How does your design express that, and where does the check actually happen?"*

The key insight this question is probing for: permissions are **per-project**, not global to a user — the same person needs different capabilities depending on which project's data they're touching, which rules out a single, flat `user.role` field entirely.

---

# 40. Role-Based Access Control: Roles, Permissions, and Project-Level Overrides

```java
public enum Permission { CREATE_ISSUE, EDIT_ISSUE, DELETE_ISSUE, TRANSITION_ISSUE, MANAGE_WORKFLOW, MANAGE_MEMBERS }

public class Role {
    private final String name;                          // "Admin", "Contributor", "Viewer"
    private final Set<Permission> permissions;
}

// The actual per-project assignment — THIS is what makes RBAC per-project, not global
public class ProjectMembership {
    private final String userId;
    private final String projectId;
    private Role role;
}
```

`ProjectMembership` is the entity that resolves §39's exact scenario: Alice has one `ProjectMembership` row for Project A (with the Admin role) and a separate one for Project B (with the Contributor role) — the same `User` object, two independent role assignments, checked against whichever project the current request actually concerns.

---

# 41. Implementing a PermissionChecker

```java
public class PermissionChecker {
    private final ProjectMembershipRepository membershipRepository;

    public void requirePermission(String userId, String projectId, Permission permission) {
        ProjectMembership membership = membershipRepository.find(userId, projectId);
        if (membership == null || !membership.getRole().getPermissions().contains(permission)) {
            throw new AccessDeniedException(userId, projectId, permission);
        }
    }
}
```

```java
// Used at the START of every write path — including inside IssueStatusContext.transitionTo (§36)
public void transitionIssue(String userId, Issue issue, WorkflowState targetState, TransitionContext ctx) {
    permissionChecker.requirePermission(userId, issue.getProjectId(), Permission.TRANSITION_ISSUE);
    new IssueStatusContext(issue, workflow).transitionTo(targetState, ctx); // §36 — only reached if the check above passed
}
```

Notice `PermissionChecker` is invoked **before** any of §36–§37's workflow logic runs — permission is a coarser-grained, earlier gate ("can this user touch this project's issues at all") than a `TransitionGuard` (§37, "is this *specific* transition allowed given the issue's current state") — two genuinely different concerns, deliberately checked at two different points, rather than one bloated authorization method trying to answer both questions at once.

---

# 42. Class Diagram: RBAC Model

```text
┌──────────┐   *        1   ┌────────────────────┐   *        1   ┌──────────┐
│   User   │───────────────>│  ProjectMembership  │<───────────────│ Project  │
├──────────┤                ├────────────────────┤                ├──────────┤
│ id       │                │ userId             │                │ id       │
│ name     │                │ projectId           │                │ ...      │
└──────────┘                │ role ──────┐        │                └──────────┘
                             └────────────┼────────┘
                                          │ *      1
                                          v
                                    ┌──────────┐   *        *   ┌────────────┐
                                    │   Role   │───────────────>│ Permission │
                                    ├──────────┤                 ├────────────┤
                                    │ name     │                 │ (enum)     │
                                    └──────────┘                 └────────────┘
```

The same `User` can appear in many `ProjectMembership` rows, each pointing at a different `Project` and, independently, a different `Role` — the diagram makes §39's requirement ("different permissions on different projects") structurally obvious rather than something you'd have to infer from a paragraph of prose.

---

# 43. Follow-up Question 9 — "When an Issue Is Updated, a Dozen Things Might Need to Happen. How Do You Design That?"

> **Interviewer:** *"An issue transitions to 'Done.' Now: notify the assignee, notify anyone watching the issue, fire any configured webhooks, write an activity-log entry, and update the search index. How do you avoid the `IssueService` becoming a giant method that knows about all five of those things?"*

This is the exact motivation for the **Observer pattern** — and the question is deliberately shaped to expose a common anti-pattern: cramming every side effect directly into the method that performs the core action, which becomes an ever-growing, tightly-coupled list every time a new side effect is needed.

---

# 44. The Observer Pattern for Issue Event Handling

```java
// events/IssueEvent.java
public sealed interface IssueEvent {
    record IssueCreated(Issue issue) implements IssueEvent { }
    record IssueTransitionedEvent(Issue issue, WorkflowTransition transition) implements IssueEvent { } // referenced in §36
    record IssueCommented(Issue issue, Comment comment) implements IssueEvent { }
}

// events/EventPublisher.java
public class EventPublisher {
    private final List<IssueEventListener> listeners = new ArrayList<>();

    public void subscribe(IssueEventListener listener) { listeners.add(listener); }

    public void publish(IssueEvent event) {
        for (IssueEventListener listener : listeners) {
            listener.onEvent(event); // each listener decides independently whether/how to react
        }
    }
}

public interface IssueEventListener {
    void onEvent(IssueEvent event);
}
```

`IssueStatusContext.transitionTo` (§36) already calls `eventPublisher.publish(...)` and nothing more — it has **zero knowledge** of notifications, webhooks, search indexing, or the activity log. Every one of those five side effects §43 listed becomes an **independent** `IssueEventListener`, registered once at startup, entirely decoupled from the code that triggers the event.

---

# 45. Implementing EventPublisher and Listeners (Notification, Webhook, Audit Log)

```java
public class NotificationListener implements IssueEventListener {
    @Override
    public void onEvent(IssueEvent event) {
        if (event instanceof IssueEvent.IssueTransitionedEvent e) {
            notifyUser(e.issue().getAssigneeUserId(), "Issue " + e.issue().getKey() + " is now " + e.transition().toState().name());
        }
        // other event types this listener cares about, handled similarly — types it doesn't care about are simply ignored
    }
}

public class WebhookListener implements IssueEventListener {
    @Override
    public void onEvent(IssueEvent event) {
        for (String webhookUrl : webhooksConfiguredFor(event)) {
            asyncHttpClient.post(webhookUrl, serialize(event)); // fire-and-forget, doesn't block the transition itself
        }
    }
}

public class AuditLogListener implements IssueEventListener {
    @Override
    public void onEvent(IssueEvent event) {
        auditLogRepository.append(new AuditEntry(event, Instant.now())); // §48 — an append-only history
    }
}
```

Adding a **sixth** side effect — say, updating a "recently active issues" dashboard widget — means writing **one new class** implementing `IssueEventListener` and registering it, touching **zero** existing code, including `IssueStatusContext` itself. This is the entire payoff §43's question was probing for, made concrete: the Observer pattern turns "add a side effect" from a modification into an addition.

---

# 46. Class Diagram: Event-Driven Side Effects

```text
┌──────────────────┐  publish   ┌────────────────┐  notifies  ┌────────────────────┐
│IssueStatusContext │───────────>│ EventPublisher  │───────────>│ IssueEventListener  │ (interface)
│  (§36)            │            ├────────────────┤            ├────────────────────┤
└──────────────────┘            │ listeners: []   │             │ + onEvent(event)   │
                                 └────────────────┘             └──────────┬─────────┘
                                                                            │ implements
                              ┌─────────────────────┬──────────────────────┼──────────────────────┐
                              v                     v                      v                      v
                     ┌──────────────────┐  ┌────────────────┐  ┌────────────────────┐  ┌────────────────────┐
                     │NotificationListener│ │ WebhookListener │  │  AuditLogListener   │  │ SearchIndexListener │
                     └──────────────────┘  └────────────────┘  └────────────────────┘  └────────────────────┘
```

Every listener implements the same one-method interface and is registered with the same `EventPublisher` — the diagram's flatness (four independent boxes hanging off one interface, none aware of the others) **is** the design goal, not a simplification of a more complex reality.

---

# 47. Modeling Comments and Attachments

```java
public class Comment {
    private final String id;
    private final String issueKey;
    private final String authorUserId;
    private String body;
    private final Instant createdAt;
    private Instant editedAt; // null if never edited
}

public class Attachment {
    private final String id;
    private final String issueKey;
    private final String uploaderUserId;
    private String fileName;
    private long fileSizeBytes;
    private String storageObjectKey; // a KEY into an object store, §-cross-reference below — the file bytes live there, not here
}
```

`Attachment.storageObjectKey` deliberately doesn't hold the file's bytes at all — it's a pointer into an object store, exactly [the file storage guide's own object-storage design](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>): a relational row is the wrong place to store a 50MB PDF, but it's exactly the right place to store a small, indexed reference to where that PDF actually lives.

---

# 48. The Activity/Audit Log: An Append-Only History of Every Change

```java
public record AuditEntry(String issueKey, String actorUserId, IssueEvent event, Instant timestamp) { }

public class AuditLogRepository {
    public void append(AuditEntry entry) { /* INSERT only — this table is NEVER updated or deleted from */ }
    public List<AuditEntry> historyFor(String issueKey) { /* SELECT ... ORDER BY timestamp */ return List.of(); }
}
```

An audit log's entire value proposition depends on it being genuinely **append-only** — the same design discipline [TinyDB's own change-event log](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) is built on: if an entry could be edited or deleted after the fact, the log could no longer be trusted as a faithful record of what actually happened, which is precisely why `AuditLogListener` (§45) only ever calls `append`, never `update` or `delete`.

---

# 49. Follow-up Question 10 — "Users Want to Build Custom Filters (JQL-Style). How Do You Design That?"

> **Interviewer:** *"JIRA lets users type something like `project = "PROJ" AND status = "In Progress" AND assignee = currentUser()`. How would you design a system that parses and executes queries like that?"*

This is a smaller, self-contained version of exactly the parser/executor problem [the TinyDB guide's own SQL layer](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) solves — a small grammar, a parser producing an expression tree, and an evaluator that runs it against real data, scoped here to filtering issues instead of general-purpose SQL.

---

# 50. A Query Language for Issues: Parsing and Executing Filters

```text
Grammar (deliberately small, mirroring the discipline of keeping a first version minimal):
  expression := condition (('AND' | 'OR') condition)*
  condition  := field operator value
  field      := 'project' | 'status' | 'assignee' | 'priority' | 'label'
  operator   := '=' | '!=' | 'IN'
  value      := STRING | 'currentUser()'
```

```java
// search/FilterExpression.java
public sealed interface FilterExpression {
    record Condition(String field, String operator, String value) implements FilterExpression { }
    record And(FilterExpression left, FilterExpression right) implements FilterExpression { }
    record Or(FilterExpression left, FilterExpression right) implements FilterExpression { }
}
```

A small, hand-rolled recursive-descent parser (the exact same technique [TinyDB's SQL parser](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) uses) turns the raw filter string into a `FilterExpression` tree — `project = "PROJ" AND status = "In Progress"` becomes `And(Condition("project","=","PROJ"), Condition("status","=","In Progress"))`, ready for §51 to evaluate against real issues.

---

# 51. Implementing a Simple Filter Expression Evaluator

```java
// search/FilterEvaluator.java
public class FilterEvaluator {
    public boolean matches(Issue issue, FilterExpression expression, String currentUserId) {
        return switch (expression) {
            case FilterExpression.Condition c -> evaluateCondition(issue, c, currentUserId);
            case FilterExpression.And a -> matches(issue, a.left(), currentUserId) && matches(issue, a.right(), currentUserId);
            case FilterExpression.Or o -> matches(issue, o.left(), currentUserId) || matches(issue, o.right(), currentUserId);
        };
    }

    private boolean evaluateCondition(Issue issue, FilterExpression.Condition c, String currentUserId) {
        String actualValue = extractField(issue, c.field());          // e.g. issue.getStatus().name()
        String expectedValue = "currentUser()".equals(c.value()) ? currentUserId : c.value();
        return switch (c.operator()) {
            case "=" -> actualValue.equals(expectedValue);
            case "!=" -> !actualValue.equals(expectedValue);
            default -> throw new UnsupportedOperationException(c.operator());
        };
    }
    // extractField(...) omitted for brevity — a simple switch over c.field() reading the matching Issue getter
}
```

This tree-walking evaluator is a perfectly reasonable implementation for evaluating a filter against issues already loaded in memory (or as a fallback when the search index, §17, is unavailable); the **production** path for a query like this against millions of issues would translate the same `FilterExpression` tree into a query against the search index (§17) instead of walking it in application code row by row — the parser and the expression tree are reusable either way, only the "how do I actually execute this" backend differs, the same separation of concerns [MiniAdmin's generic CRUD layer](<Build Your Own CRUD Admin Framework From Scratch — MiniAdmin Step-by-Step Guide.md>) already demonstrated between a query's *shape* and *where it actually runs*.

---

# 52. Follow-up Question 11 — "Two Users Edit the Same Issue at the Same Time. What Happens?"

> **Interviewer:** *"Alice and Bob both open PROJ-123. Alice changes the priority. Bob, a second later, changes the assignee — but he loaded the issue before Alice's change was saved. What happens when Bob saves?"*

This is a direct, textbook **lost-update** race — exactly [the locking guide's own opening scenario](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>), just with an issue's fields instead of inventory stock. The fix is the same one that guide derives in full: **optimistic locking** via a version field, which §26 already included on `Issue` for exactly this reason.

---

# 53. Optimistic Locking for Issue Updates

```java
public void updateIssue(String issueKey, IssueUpdateRequest request, long expectedVersion) throws SQLException {
    int rowsUpdated = jdbcTemplate.update(
        "UPDATE issue SET priority = ?, assignee_user_id = ?, version = version + 1 " +
        "WHERE key = ? AND version = ?", // the version check IS the entire concurrency control mechanism
        request.priority(), request.assigneeUserId(), issueKey, expectedVersion
    );
    if (rowsUpdated == 0) {
        throw new OptimisticLockException(issueKey); // Bob's save is REJECTED, not silently overwritten or silently accepted
    }
}
```

Bob's `PATCH` request (§14) must include the `version` he last read (typically returned in the `GET` response and round-tripped by the client) — if Alice's update already bumped the version, Bob's conditional `UPDATE` matches zero rows, and the API returns a `409 Conflict` the client surfaces as "this issue changed since you loaded it, please refresh" — never a silent, undetected overwrite of Alice's change. [The locking guide's full treatment](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>) covers exactly why optimistic (rather than pessimistic, lock-and-block) locking is the right default here: issue edits are infrequent and low-contention compared to that guide's flash-sale example, exactly the profile optimistic locking is best suited for.

---

# 54. Follow-up Question 12 — "How Do You Generate Unique, Human-Readable Issue Keys (PROJ-123) Without Collisions at Scale?"

> **Interviewer:** *"Every issue needs a key like PROJ-123 — unique within the project, sequential-looking, and generated correctly even if two people create an issue in the same project at the exact same instant, possibly on different application servers. How?"*

A naive "read the current max number for this project, add one" approach has the exact same lost-update race as §52 — two concurrent issue creations could both read the same max and both compute the same "next" number, producing two issues both claiming to be `PROJ-457`.

---

# 55. Distributed ID Generation for Issue Keys

```java
public class IssueKeyGenerator {
    private final DataSource dataSource;

    /** Uses an atomic, database-level counter increment — the SQL server resolves the race, not the application. */
    public String nextKey(String projectKey) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "UPDATE project_counter SET next_number = next_number + 1 " +
                 "WHERE project_key = ? RETURNING next_number")) { // atomic increment-and-return, in ONE statement
            ps.setString(1, projectKey);
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return projectKey + "-" + rs.getLong("next_number");
            }
        }
    }
}
```

The critical detail: incrementing and reading the counter happens in **one atomic database statement** (`UPDATE ... RETURNING`), not as a separate `SELECT` followed by an `UPDATE` — which would reopen exactly the same race §54 warned about, just moved into this class instead of avoiding it. This pushes the concurrency problem onto the database's own row-level atomicity guarantee (conceptually the same single-row atomic update guarantee [TinyDB's row-level locking](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) provides), rather than trying to solve it in application code — the correct instinct any time a strictly-increasing, collision-free counter is needed under real concurrency.

---

# 56. Follow-up Question 13 — "This Now Has 10 Million Issues and 100,000 Concurrent Users. What Breaks First, and How Do You Fix It?"

> **Interviewer:** *"Let's fast-forward. The product succeeded. What's the first thing that falls over at this scale, and what specifically do you do about it — not in general terms, be specific."*

This is the interview's final escalation — testing whether a candidate can reason about **which** bottleneck hits first, in order, rather than reciting a generic list of "add caching, add sharding, add replicas" with no sense of sequencing or root cause.

---

# 57. Database Sharding Strategy

The single relational database (§15) is the first thing to strain — specifically, write throughput and index size on the `issue` table. **Sharding by `tenant_id`** (§22's multi-tenancy column, reused directly here) is the natural first cut: since no query ever legitimately needs to join across two different tenants' data (they're different companies entirely), routing each tenant's data to a specific shard based on a hash of `tenant_id` scales write throughput roughly linearly with shard count, with **zero** cross-shard queries ever required for normal operation — the best-case scenario for sharding, because the natural data-isolation boundary (tenant) and the natural query-isolation boundary (a query never spans tenants) are the same boundary.

---

# 58. Read Replicas and CQRS-Style Read/Write Separation

Even within one shard, read traffic (loading boards, viewing issues) vastly outweighs write traffic (creating/updating issues) — a classic **read replica** setup, where writes go to a primary and reads are served from one or more asynchronously-replicated followers, absorbs the read load without adding write-side complexity. This is architecturally the same **leader-follower replication** [TinyDB's own replication design](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) implements — the same tradeoff applies too: a read from a replica can be milliseconds stale, which is entirely acceptable for viewing a board but must be handled carefully for anything that just wrote and immediately needs to read its own write (the same "read-your-writes" concern that guide's replication section names explicitly).

---

# 59. Rate Limiting the API

At 100,000 concurrent users, a single misbehaving client (a buggy integration polling too aggressively, or a genuine abuse case) can degrade service for everyone else sharing the same infrastructure — a **rate limiter** at the API gateway (§24), keyed by tenant and/or API token, rejects requests past a configured threshold with a `429 Too Many Requests` **before** they ever reach a service, protecting the Issue/Project/Workflow services from load they were never sized to handle from one bad actor.

---

# 60. Full Worked Example: Creating and Transitioning an Issue End to End

```text
POST /projects/PROJ/issues { title, description, type: STORY, priority: HIGH }
  -> PermissionChecker.requirePermission(userId, "PROJ", CREATE_ISSUE)                    (§41)
  -> IssueKeyGenerator.nextKey("PROJ") -> "PROJ-458"                                       (§55)
  -> Issue saved to the sharded relational DB, tenant_id set, version = 0                  (§26, §57)
  -> EventPublisher.publish(IssueCreated(issue))                                           (§44)
       -> SearchIndexListener indexes it                                                   (§17)
       -> AuditLogListener appends an entry                                                (§48)

POST /issues/PROJ-458/transitions { toState: "IN_PROGRESS" }
  -> PermissionChecker.requirePermission(userId, "PROJ", TRANSITION_ISSUE)                 (§41)
  -> IssueStatusContext.transitionTo(IN_PROGRESS, ctx)                                      (§36)
       -> Workflow.isValidTransition(TODO, IN_PROGRESS) -> true                             (§35)
       -> guards checked (none configured for this transition, in this example)             (§37)
       -> issue.setStatus(IN_PROGRESS), UPDATE ... WHERE version = ? (optimistic lock)      (§53)
       -> EventPublisher.publish(IssueTransitionedEvent(issue, transition))                 (§44)
            -> NotificationListener notifies the assignee                                   (§45)
            -> WebSocket pub/sub pushes a live board update to everyone viewing PROJ's board (§19)
            -> SearchIndexListener updates the index                                        (§17)
```

Every mechanism this guide built — permissions, key generation, the workflow engine, optimistic locking, and the full event fan-out — appears somewhere in these two requests, which is the clearest possible demonstration that the pieces genuinely compose into one coherent system rather than being independent exercises.

---

# 61. Final Architecture Diagram

```text
                          Client
                              |
              API Gateway (auth, rate limiting, §59)
                              |
        +---------+---------+---------+---------+
        v         v         v         v         v
    Project   Issue     Workflow  Notification  Search
    Service   Service    Service    Service      Service
        |         |          |
        +---------+----------+
                   |
     PermissionChecker (§41) gates every write
                   |
     IssueStatusContext (§36) -- Workflow (§35) -- TransitionGuards (§37)
                   |
        Sharded relational DB, by tenant_id (§22, §57)
        + read replicas (§58)
                   |
         publishes IssueEvents (§44)
                   |
        +----------+----------+----------+
        v          v          v          v
  SearchIndexer  Notification  Webhook  AuditLog
  (§17)          (§45)         (§45)    (§45, §48)
```

---

# 62. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **State** | Issue status transitions, validated against a per-project `Workflow` | §35-§36 |
| **Strategy** | `TransitionGuard` implementations, pluggable per transition | §37 |
| **Observer** | Fan-out of side effects (notifications, webhooks, audit log, search indexing) on any issue event | §44-§46 |
| **Composition over Inheritance** | Epic/Story/SubTask as one `Issue` class with a `type` field, not a class hierarchy | §29 |
| **Repository** (implicit) | Data access abstracted behind repository interfaces, never raw SQL scattered through business logic | Throughout Part 3 onward |

---

# 63. SOLID Principles Applied

| Principle | Where it holds |
|---|---|
| **S**ingle Responsibility | `IssueStatusContext` only validates and performs transitions; `PermissionChecker` only checks permissions; `EventPublisher` only fans out events — none of them do more than one job |
| **O**pen/Closed | Adding a new `IssueEventListener` (§45) or a new `TransitionGuard` (§37) is purely additive — no existing class changes |
| **L**iskov Substitution | Every `TransitionGuard`/`IssueEventListener` implementation is fully interchangeable from its caller's point of view | 
| **I**nterface Segregation | `IssueEventListener` and `TransitionGuard` are each single-method interfaces — no implementation is forced to support behavior it doesn't need |
| **D**ependency Inversion | `IssueStatusContext` depends on the `Workflow`/`TransitionGuard` abstractions, never on a specific project's concrete configuration |

---

# 64. Common Mistakes

- **Mistake 1 — A `List<Issue>` field directly on `Project` (§28).** Looks natural in a diagram, catastrophic at real scale — always query issues by project ID through a repository, never hold them as an in-memory collection on the parent.
- **Mistake 2 — A hard-coded `Status` enum shared across all projects (§34).** Directly fails the "customizable workflow per project" requirement the moment two projects need different statuses.
- **Mistake 3 — Cramming every side effect into the transition method itself (§43).** Turns a focused state-machine operation into an ever-growing, tightly-coupled god-method every time a new notification channel is added.
- **Mistake 4 — Checking permissions *after* performing a mutation, or skipping the check for "trusted" internal callers.** Every write path must call `PermissionChecker` first, with no exceptions, or a future caller (or a future bug) silently bypasses authorization.
- **Mistake 5 — A read-then-write (not atomic) issue key generator (§54).** Reopens the exact concurrent-collision problem it was meant to solve; the increment and read must be one atomic database operation.
- **Mistake 6 — Treating the search index as the source of truth.** It's a derived, eventually-consistent projection (§17) — the relational database (§15) is authoritative, and the index must always be rebuildable from it.

---

# 65. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| Workflow engine (§35-§37) | Only configured transitions succeed; a missing/failing guard blocks the transition | Build a small test `Workflow` with two states and a `RequiresQaApprovalGuard`, assert both the allowed and blocked paths |
| Permission checks (§41) | A user without the required role on a project is denied, even if they have it on a different project | Two `ProjectMembership` rows for the same user, different projects, different roles — assert per-project isolation |
| Event fan-out (§44-§45) | Every registered listener receives every published event; a listener throwing an exception doesn't prevent others from running | Register several fake listeners, publish one event, assert all were invoked |
| Optimistic locking (§53) | A concurrent update with a stale version is rejected, not silently overwritten | The same concurrent-load harness style as [the locking guide's §48](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>) |
| Issue key generation (§55) | No two concurrently-created issues in the same project ever receive the same key | Many threads calling `nextKey` concurrently for the same project, asserting every returned key is unique |
| Filter evaluator (§50-§51) | A parsed `AND`/`OR` expression correctly matches/excludes issues | Build a small in-memory issue set, assert filter results against hand-computed expected matches |

---

# 66. Suggested Future Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Custom fields per project | Projects define their own issue fields beyond the fixed set (§26) | An EAV-style extension, the same technique [MiniAdmin's Custom Object builder](<Build Your Own CRUD Admin Framework From Scratch — MiniAdmin Step-by-Step Guide.md>) uses for user-defined schema |
| Automation rules ("when X, do Y") | User-configurable triggers, e.g. "when an issue is labeled 'urgent', notify the team lead" | A rules engine subscribing to the same `IssueEvent` stream (§44) as the built-in listeners |
| Time tracking and reporting (burndown charts) | Sprint velocity, cycle time, and other analytics | A separate read-optimized analytics store, fed by the same event stream, mirroring §17's search-indexer pattern |
| Cross-project epics and dependencies | An epic (or a "blocks"/"is blocked by" link) spanning multiple projects | Relaxes §28's project-scoped assumption — a genuinely bigger design change, not just an additive one |
| Fine-grained field-level permissions | "Contributors can edit description but not priority" | Extends §40's `Permission` enum to be field-aware, checked inside the update path rather than only at the operation level |
| Real-time collaborative editing of descriptions | Multiple users editing the same issue's description simultaneously, like a shared document | Operational transforms or CRDTs — a substantially more complex mechanism than §19's simple change-broadcast |

---

# 67. Progressive Interview Question Set

**Level 1 — Requirements and domain modeling**
1. What functional requirement, if changed, would most change your high-level architecture — and why?
2. Why isn't an Epic modeled as a separate Java class from a Story or Bug?

**Level 2 — High-level design**
3. Justify the service boundary between the Issue Service and the Workflow Service — why not combine them?
4. Walk through what happens, system-wide, when one issue changes status — name every downstream consumer of that event.

**Level 3 — Low-level design**
5. Why can't `Issue.status` be a plain enum if workflows are customizable per project?
6. Explain how a `TransitionGuard` differs from a `PermissionChecker` check, and why both exist as separate mechanisms.

**Level 4 — Concurrency and scale**
7. Trace exactly what happens when two users edit the same issue concurrently, from both users' perspective.
8. Why must issue key generation be one atomic database operation, not a read followed by a write?
9. Justify sharding by `tenant_id` specifically — why is this a favorable sharding key for this domain, and what would make a sharding key *unfavorable*?

**Final challenge:** A customer wants issues to support **field-level history** — not just "the issue changed" (§48's audit log), but "priority changed from Low to High by Alice at 3:04pm, and separately, assignee changed from Bob to Carol at 3:06pm" — queryable per field. Design the change to `AuditEntry` (§48) and to whichever listener populates it, and explain what tradeoff (storage volume vs. query granularity) this enhancement makes explicit that the original coarse-grained audit log didn't have to confront.

---

# 68. Final Takeaway

Every escalating follow-up question in this guide pushed on the same underlying discipline: **a good answer to "design X" is never a single diagram — it's a chain of justified decisions, each one defensible against "why not something simpler."** A fixed status enum is simpler than a data-driven workflow engine (§34–§35) — until a customizable-workflow requirement (§7) makes it wrong. A `List<Issue>` field on `Project` is simpler than a repository query (§28) — until scale makes it wrong. A synchronous side-effect call is simpler than the Observer pattern (§43–§44) — until the fifth side effect makes it unmaintainable. None of these "more complex" answers were reached for by default; each was justified by a specific requirement or a specific follow-up question exposing exactly why the simpler version breaks. That's the actual skill a JIRA-style system-design interview is testing — not whether you've memorized what JIRA looks like, but whether you can derive its design decisions, one defensible step at a time, the way this guide just did.


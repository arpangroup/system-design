# Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a workflow automation system — something like Zapier, n8n, or a lightweight Temporal/Airflow. Cover both high-level design (services, data flow, scaling) and low-level design (class diagrams, core algorithms), and make sure your design follows SOLID principles. Be ready to justify every decision when I push back."*
>
> This guide is structured exactly as that interview unfolds: a requirements-gathering phase, a high-level architecture built up decision by decision, a low-level class design deep dive, and a sequence of escalating **follow-up questions** — each answered with real reasoning and, where it matters, real Java code — not a single static diagram presented as if no one ever questioned it.

---

# 1. What We Are Building

We are building **MiniFlow** — a workflow automation engine covering:

- **Functional requirements**: user-defined workflows made of triggers and steps, conditional branching, parallel fan-out/fan-in, loops, retries, human-approval steps, sub-workflows, and a pluggable integration ("connector") layer.
- **High-level design**: service decomposition, API shape, data storage choices, trigger ingestion (webhook/schedule/event), and asynchronous, durable execution.
- **Low-level design**: a DAG-based workflow definition model, a genuinely pluggable execution engine (the State pattern for run/step lifecycle), a Strategy-based connector layer, an expression evaluator for step-to-step data mapping, and a Saga-style compensation mechanism for partial failures.
- **Reliability concerns**: crash-safe durable execution via an event log, retries with idempotency, dead-lettering, workflow versioning for in-flight runs, and partitioned scaling.

```text
                    Trigger sources (webhook / cron / internal events / manual)
                                          |
                                   Trigger Service
                                          |
                                Orchestration Engine  <----> Definition Store (versioned DAGs)
                                          |                          |
                          +---------------+----------------+   Event Log (durable, per-run)
                          v               v                v
                    Worker Pool     Worker Pool      Worker Pool
                   (HTTP action)   (email action)   (custom code)
                          |               |                |
                          +-------+-------+-------+--------+
                                  v
                         External systems / connectors
                     (Slack, Stripe, Salesforce, plain HTTP, ...)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Gather and state functional/non-functional requirements for an ambiguous "design a workflow automation tool" prompt, before designing anything.
- Model a workflow as a **DAG** (directed acyclic graph) that is simultaneously human-editable (a DSL) and machine-executable, and validate it (cycle detection, reachability) before it ever runs.
- Design an execution engine whose run/step lifecycle is driven by the **State pattern**, not a hard-coded status `if/else` chain — and make that engine **crash-safe** using an append-only event log, the same durability discipline a database's write-ahead log provides.
- Design a genuinely pluggable **connector layer** (the Strategy pattern plus a registry) so adding a new integration never requires touching the orchestration engine.
- Explain, with real code, how retries avoid duplicate side effects (idempotency keys), how a multi-day human-approval step doesn't block a thread, and how a partially-completed workflow rolls back its real-world side effects (the Saga pattern).
- Reason about what breaks first at millions of runs per day across thousands of tenants, and name the specific techniques (partitioning, backpressure, autoscaling) that address each bottleneck.

---

# 3. Why This Matters (The Interview, Framed)

A workflow automation system is a favorite senior/staff design question for a specific reason: it looks, on the surface, like "just a task queue with extra steps," but a candidate who treats it that way misses almost everything that makes it hard. The question forces movement across the same three levels every serious system-design interview probes:

- **Requirements-driven scoping** — "workflow automation" spans everything from a simple linear Zapier-style "if this, then that" to a Temporal-style durable-execution runtime for long-lived business processes. A strong candidate narrows this explicitly, the same discipline [the file storage guide's opening](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) and [the JIRA-style guide's opening](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) both insist on.
- **High-level architecture** — trigger ingestion, a durable execution engine, a worker pool, and a connector layer, each with real scaling and failure-mode reasoning behind it, not a diagram memorized without understanding.
- **Low-level, code-level design** — this is where the question actually separates candidates: translating "steps can run in parallel, retry on failure, and sometimes wait for a human" into a real class model that survives a crash mid-run without corrupting or duplicating anything. Most of the genuinely hard engineering in a workflow engine lives exactly here.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and pattern implementations (State, Strategy, Command). |
| Definition store | A relational database (Postgres-style, or [TinyDB](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) from the companion guide) | Workflow definitions are structured, versioned, and queried by ID — a natural fit for real transactions (§58). |
| Execution event log | An append-only, durable log (Kafka-style, or the WAL discipline from the [TinyDB Storage Engine guide](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>)) | Crash-safe resumability (§21-§23) needs exactly the "durable before acted upon" guarantee a write-ahead log already provides. |
| Task dispatch | A durable work queue (SQS/Kafka-style) | Decouples "a step is ready to run" from "a specific worker happens to be free right now" (§13, §61-§62). |
| Worker pool | A bounded thread/process pool per connector type | The exact core-queue-max-reject shape the [Build Your Own Executor Framework guide](<Build Your Own Executor Framework From Scratch — A Java Concurrency Step-by-Step Guide.md>) already builds from scratch, reused here per connector (§39). |
| Cache | An in-memory key-value store (Redis-style) | Hot-path reads (a workflow's current definition, a tenant's rate-limit counters) avoid a database round trip on every step dispatch. |

---

# 5. Project Structure

```text
miniflow/
├── src/main/java/com/example/miniflow/
│   ├── definition/
│   │   ├── WorkflowDefinition.java, StepDefinition.java, Edge.java  // §17
│   │   ├── WorkflowValidator.java                                   // §18
│   │   └── WorkflowDefinitionRegistry.java                          // §58
│   ├── execution/
│   │   ├── WorkflowRun.java, StepExecution.java                     // §30
│   │   ├── RunStatus.java, StepStatus.java (State pattern)          // §33-§34
│   │   ├── OrchestrationEngine.java                                 // §20, §23, §44-§46
│   │   └── ExpressionEvaluator.java                                 // §42
│   ├── eventlog/
│   │   ├── WorkflowEvent.java (sealed)                               // §22
│   │   └── EventLog.java                                             // §22-§23
│   ├── connectors/
│   │   ├── ActionExecutor.java, ActionExecutorRegistry.java          // §37
│   │   ├── HttpActionExecutor.java, EmailActionExecutor.java         // §38
│   │   └── RetryPolicy.java, ExponentialBackoffRetryPolicy.java      // §48
│   ├── triggers/
│   │   ├── ScheduleTrigger.java, TriggerScheduler.java                // §26
│   │   └── WebhookTrigger.java                                       // §27
│   ├── saga/
│   │   └── CompensableStep.java                                      // §55
│   └── idempotency/
│       └── IdempotencyKey.java                                       // §50
└── src/test/java/com/example/miniflow/
    ├── OrchestrationEngineCrashRecoveryTest.java
    ├── RetryAndIdempotencyTest.java
    └── SagaCompensationTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"Is this single-tenant or a multi-tenant SaaS product? Do workflows need to support long-running, multi-day steps (like waiting for a human approval), or is everything expected to finish in seconds? What's the expected scale — thousands or millions of runs a day? Do users write workflows in a visual builder, a YAML/JSON DSL, or both? Do we need exactly-once execution guarantees for side-effecting steps, or is at-least-once with idempotency acceptable?"*

Exactly as [the file storage guide's opening](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) argues, jumping straight to boxes-and-arrows without narrowing "design a workflow automation system" is the first mistake a candidate can make. The answers determine whether durable, multi-day execution (§21, §52-§53) and Saga-style compensation (§54-§56) are core requirements or premature complexity.

For this guide, we settle on a concrete, realistic scope: **multi-tenant SaaS**, workflows that **may run for days** (human approval steps are a first-class feature), targeting **millions of runs per day**, definitions authored as a **JSON/YAML DSL**, with **at-least-once execution plus idempotency** as the realistic, honestly-stated guarantee (exactly-once execution of arbitrary external side effects is not achievable in general — §50 explains precisely why, and what idempotency keys actually buy instead).

---

# 7. Functional Requirements

- Define a **workflow** as a graph of **steps** connected by **edges**, optionally with conditions on those edges.
- Support step types: an **action** step (call a connector — HTTP, email, Slack, custom code), a **condition** step (branch), a **parallel** step (fan-out), a **join** (fan-in), a **for-each** step (loop over a collection), an **approval** step (pause for a human), and a **sub-workflow** step (compose another workflow as one step).
- Trigger a workflow run via a **webhook**, a **schedule** (cron), an **internal event**, or a **manual** invocation.
- Retry a failed step automatically, with a configurable backoff policy, without duplicating the step's real-world side effects.
- Let a workflow **roll back** its already-completed side effects if a later step fails permanently (compensation).
- Let users add **new connectors** (integrations) without modifying the orchestration engine itself.
- Let a **new version** of a workflow definition be published without breaking runs that are already in flight on the old version.
- Provide a **run history** view: every step's status, inputs, outputs, and timing, for debugging a specific failed run.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Durability** | A crash of any worker or orchestrator process must never silently lose or duplicate a run's progress | The event log and replay-based recovery (§21-§23) |
| **Scalability** | Millions of runs/day across thousands of tenants, without a redesign | Partitioning by run ID (§60), per-connector backpressure (§61), worker autoscaling (§62) |
| **Extensibility** | Adding a new step type or connector must never require modifying the orchestration engine | The Strategy pattern for connectors (§38), the sealed `StepDefinition` hierarchy for step types (§30) |
| **Resilience to flaky externals** | A transient failure calling Slack or Stripe must not fail the whole run, and a retry must not double-charge anyone | Exponential backoff (§49), idempotency keys (§50), dead-lettering (§51) |
| **Long-running steps** | An approval step waiting three days must not hold a thread, a connection, or any other scarce resource for those three days | The `WAITING` state and external-callback resume (§52-§53) |

---

# 9. Follow-up Question 1 — "What Are the Core Nouns in This System, Before We Draw Any Boxes?"

> **Interviewer:** *"Before services and databases — what actually exists in this system, and how does it relate?"*

Exactly the same deliberate pivot [the JIRA-style guide's own Follow-up 1](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) makes: a service boundary or schema that doesn't reflect a clearly-understood domain model invites painful rework later. §10 answers it directly; §17 and §30 build every one of these entities as real code.

---

# 10. Identifying the Core Domain Entities

| Entity | Represents | Key relationships |
|---|---|---|
| **WorkflowDefinition** | The versioned, immutable-once-published blueprint of a workflow | Made of `StepDefinition`s connected by `Edge`s |
| **StepDefinition** | One node in the graph — an action, condition, parallel, join, for-each, approval, or sub-workflow | Has zero or more outgoing `Edge`s |
| **Edge** | A directed connection between two steps, optionally guarded by a condition | Connects exactly two `StepDefinition`s |
| **WorkflowRun** | One executing (or completed) instance of a `WorkflowDefinition` | Has many `StepExecution`s, pinned to one definition version |
| **StepExecution** | The runtime record of one step within one run — its status, input, output, attempt count | Belongs to exactly one `WorkflowRun` |
| **Trigger** | The thing that starts a `WorkflowRun` — webhook, schedule, event, or manual | Bound to exactly one `WorkflowDefinition` |
| **Connector / ActionExecutor** | The pluggable code that actually performs one action step's real-world effect | Selected by a step's declared type, via a registry (§38) |
| **WorkflowEvent** | One immutable, durable fact appended to a run's event log — "step X started," "step X succeeded with output Y" | The source of truth `WorkflowRun`/`StepExecution` state is *derived from* (§22) |

Every section from §11 onward either builds infrastructure **around** these entities or builds the entities **themselves** as real, working class designs — nothing introduced later is untraceable back to this table.

---

# 11. High-Level Architecture Overview

```text
                                Client (workflow builder UI / API)
                                          |
                             API Gateway / Load Balancer
                                          |
        +---------------+---------------+---------------+---------------+
        v               v               v               v               v
   Definition       Trigger      Orchestration        Worker          Connector
    Service         Service          Engine            Pool            Service
        |               |               |               |               |
        +---------------+-------+-------+-------+-------+---------------+
                                v               v
                     Definition Store      Event Log (durable, append-only)
                     (relational)          + a durable work queue for dispatch
```

Six cooperating pieces, each with one job: **Definition Service** owns publishing and validating `WorkflowDefinition`s; **Trigger Service** owns webhook ingestion and cron scheduling; **Orchestration Engine** owns walking a run's graph and deciding what's next; **Worker Pool** owns actually executing one step's connector call; **Connector Service** owns the pluggable integration layer; the **Event Log** is the durable spine everything else is built on top of.

---

# 12. Follow-up Question 2 — "Monolith or Microservices? Justify It."

> **Interviewer:** *"You've drawn six boxes. Are those six separate deployable services, or one application with six modules? Why?"*

The honest answer: **a modular monolith at first, split only where scaling or failure-isolation genuinely demands it.** The Worker Pool is the one component with a real, independent reason to scale separately and fail independently from the rest — a burst of Slack-connector calls should never starve the orchestration engine's ability to keep walking other runs' graphs forward. Everything else (Definition Service, Trigger Service, Orchestration Engine) can start as modules inside one process talking to the same database and event log, and split out later purely as a scaling decision, not an architectural requirement decided on day one — the same "don't reach for microservices reflexively" discipline [the JIRA-style guide's Follow-up 2](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) argues for issue tracking.

---

# 13. Service Decomposition: Definition, Trigger, Orchestration, Worker, Connector

- **Definition Service** — CRUD and versioning for `WorkflowDefinition`s; runs `WorkflowValidator` (§18) before ever marking a version publishable.
- **Trigger Service** — ingests webhooks (§27), runs the cron scheduler (§26), and subscribes to the internal event bus; on a fire, calls the Orchestration Engine's `startRun`.
- **Orchestration Engine** — the graph walker (§20, §44-§46); on each step completion, decides which step(s) become ready next, and enqueues their dispatch.
- **Worker Pool** — one bounded thread/process pool per connector *type* (an HTTP pool, an email pool, a custom-code sandbox pool), so one slow connector can never starve another — precisely the isolated-pool argument the [Executor Framework guide](<Build Your Own Executor Framework From Scratch — A Java Concurrency Step-by-Step Guide.md>) makes for isolating slow tasks from fast ones.
- **Connector Service** — the `ActionExecutor` registry (§38) workers call into; this is the one piece third-party integration authors ever touch.

---

# 14. API Design: The Core REST Endpoints

```text
POST   /workflows                         Create a new WorkflowDefinition (draft)
POST   /workflows/{id}/versions           Publish a new version (validated, §18)
GET    /workflows/{id}                    Fetch the current (or a specific) version

POST   /workflows/{id}/runs               Manually start a run
GET    /runs/{runId}                      Run status + full step history
POST   /runs/{runId}/steps/{stepId}/resume  Resume a WAITING approval step (§53)
POST   /runs/{runId}/cancel               Cancel an in-flight run

POST   /webhooks/{triggerId}              Webhook ingestion endpoint (§27)
```

---

# 15. Data Storage Choices: Relational Metadata Plus a Durable Event Log

Two fundamentally different kinds of data need two fundamentally different storage shapes: `WorkflowDefinition`s are structured, versioned, and queried by ID and tenant — a relational store, with real foreign keys and transactions, is the right fit. A run's execution history is instead an append-only sequence of immutable facts ("step X started," "step X succeeded") — exactly the shape a **durable, ordered log** (not a mutable row that gets overwritten in place) is built for, and exactly the shape §22's Event Sourcing model uses. Deriving a `WorkflowRun`'s *current* state by replaying its event log, rather than storing "current state" as the only source of truth, is what makes crash recovery (§23) and human-approval resume (§53) both reuse the *same* mechanism instead of needing two different ones.

---

# 16. Follow-up Question 3 — "How Do You Represent a Workflow Definition So It's Both Human-Editable and Machine-Executable?"

> **Interviewer:** *"Users need to author these things — visually, or by hand in a config file. But your engine also needs to execute them deterministically. How do those two needs coexist in one model?"*

The standard answer, used by every real system in this space (GitHub Actions, AWS Step Functions, Temporal, n8n): a **declarative DSL** (JSON or YAML) that serializes directly into an in-memory **DAG** — a directed graph of steps and edges, required to be acyclic so "what runs next" is always well-defined and a workflow can never infinitely loop on itself by construction. §17 defines the class model; §18 builds the validator that rejects a cycle *before* a definition is ever published, not after a run gets stuck in one.

---

# 17. The Workflow Definition Model: A DAG-Based DSL

```java
public sealed interface StepDefinition permits ActionStep, ConditionStep, ParallelStep, JoinStep, ForEachStep, ApprovalStep, SubWorkflowStep {
    String id();
    List<Edge> outgoingEdges();
}

public record ActionStep(String id, String connectorType, Map<String, String> inputTemplate, List<Edge> outgoingEdges) implements StepDefinition { }
public record ConditionStep(String id, String expression, List<Edge> outgoingEdges) implements StepDefinition { }        // §44
public record ParallelStep(String id, List<Edge> outgoingEdges) implements StepDefinition { }                            // §45 -- fan-out
public record JoinStep(String id, int expectedIncomingCount, List<Edge> outgoingEdges) implements StepDefinition { }     // §45 -- fan-in
public record ForEachStep(String id, String collectionExpression, StepDefinition body, List<Edge> outgoingEdges) implements StepDefinition { } // §46
public record ApprovalStep(String id, String approverGroup, List<Edge> outgoingEdges) implements StepDefinition { }       // §52-§53
public record SubWorkflowStep(String id, String subWorkflowId, List<Edge> outgoingEdges) implements StepDefinition { }
```

```java
public record Edge(String fromStepId, String toStepId, String condition /* nullable -- null means unconditional */) { }

public record WorkflowDefinition(
        String workflowId,
        int version,
        String triggerType,               // "webhook" | "schedule" | "event" | "manual" -- §25
        Map<String, StepDefinition> stepsById,
        String startStepId
) { }
```

A `sealed interface` for `StepDefinition`, exhaustively matched everywhere the engine has to decide "how do I execute this kind of step" (§20's dispatcher, §44's graph walker), is the same compiler-enforced-completeness discipline this session's other guides lean on for AST dispatch — adding an eighth step type is then a compile error at every `switch` until every one of them is updated, never a silent runtime gap.

---

# 18. Validating a Workflow Definition: Cycle Detection and Reachability

```java
public final class WorkflowValidator {

    public List<String> validate(WorkflowDefinition definition) {
        List<String> errors = new ArrayList<>();

        if (!definition.stepsById().containsKey(definition.startStepId())) {
            errors.add("startStepId '" + definition.startStepId() + "' does not exist");
            return errors; // nothing else can be checked meaningfully without a valid start
        }

        Set<String> reachable = reachableFrom(definition.startStepId(), definition);
        for (String stepId : definition.stepsById().keySet()) {
            if (!reachable.contains(stepId)) {
                errors.add("Step '" + stepId + "' is unreachable from the start step");
            }
        }

        if (hasCycle(definition)) {
            errors.add("Workflow graph contains a cycle -- a DAG cannot contain one by definition");
        }

        for (StepDefinition step : definition.stepsById().values()) {
            for (Edge edge : step.outgoingEdges()) {
                if (!definition.stepsById().containsKey(edge.toStepId())) {
                    errors.add("Edge from '" + step.id() + "' targets unknown step '" + edge.toStepId() + "'");
                }
            }
        }
        return errors;
    }

    private Set<String> reachableFrom(String startStepId, WorkflowDefinition definition) {
        Set<String> visited = new HashSet<>();
        Deque<String> stack = new ArrayDeque<>(List.of(startStepId));
        while (!stack.isEmpty()) {
            String current = stack.pop();
            if (!visited.add(current)) continue;
            StepDefinition step = definition.stepsById().get(current);
            if (step == null) continue;
            for (Edge edge : step.outgoingEdges()) stack.push(edge.toStepId());
        }
        return visited;
    }

    private boolean hasCycle(WorkflowDefinition definition) {
        Set<String> visiting = new HashSet<>();
        Set<String> done = new HashSet<>();
        for (String stepId : definition.stepsById().keySet()) {
            if (!done.contains(stepId) && dfsHasCycle(stepId, definition, visiting, done)) return true;
        }
        return false;
    }

    private boolean dfsHasCycle(String stepId, WorkflowDefinition definition, Set<String> visiting, Set<String> done) {
        if (visiting.contains(stepId)) return true;   // currently on the DFS stack -- a back edge, i.e. a cycle
        if (done.contains(stepId)) return false;       // already fully explored, known cycle-free from here
        visiting.add(stepId);
        StepDefinition step = definition.stepsById().get(stepId);
        if (step != null) {
            for (Edge edge : step.outgoingEdges()) {
                if (dfsHasCycle(edge.toStepId(), definition, visiting, done)) return true;
            }
        }
        visiting.remove(stepId);
        done.add(stepId);
        return false;
    }
```

The classic three-color DFS (`visiting` = gray, `done` = black, unvisited = white) is what makes cycle detection correct on a graph with shared sub-paths (two steps both feeding into one join step, say) rather than a simple visited-set check, which would falsely flag re-visiting a node through a *different* path as a cycle. A definition failing either check here is rejected at publish time (§14's `POST /workflows/{id}/versions`) — never at run time, where a stuck run is a far more expensive way to discover the same bug.

---

# 19. Follow-up Question 4 — "Walk Me Through Executing a Workflow, Step by Step, In Memory"

> **Interviewer:** *"Forget durability for a second. Given a validated `WorkflowDefinition` and a starting input, how does your engine actually walk the graph and produce a result?"*

This is deliberately asked before durability, because the *shape* of the walk has to be right before worrying about surviving a crash mid-walk. §20 builds the naive, in-memory-only version; §21's follow-up is exactly the question that breaks it.

---

# 20. The Naive In-Memory Execution Loop (and Why It's Not Enough)

```java
public final class NaiveOrchestrator {

    private final ActionExecutorRegistry registry; // §37

    public NaiveOrchestrator(ActionExecutorRegistry registry) { this.registry = registry; }

    public Map<String, Object> run(WorkflowDefinition definition, Map<String, Object> triggerInput) {
        Map<String, Object> context = new HashMap<>();
        context.put("trigger", triggerInput);

        Deque<String> ready = new ArrayDeque<>(List.of(definition.startStepId()));
        Set<String> completed = new HashSet<>();

        while (!ready.isEmpty()) {
            String stepId = ready.poll();
            StepDefinition step = definition.stepsById().get(stepId);
            Object output = execute(step, context); // §37's registry dispatch, by step type
            context.put(stepId, output);
            completed.add(stepId);

            for (Edge edge : step.outgoingEdges()) {
                if (edge.condition() == null || evaluateCondition(edge.condition(), context)) {
                    ready.add(edge.toStepId());
                }
            }
        }
        return context;
    }

    // execute(...)/evaluateCondition(...) elided here -- §37 and §41 build them for real.
}
```

This is correct for a straight-line or simple branching workflow, entirely in one process, with no crash to survive. It is also, deliberately, missing everything that makes a *production* workflow engine hard: nothing here is written to disk, so a crash mid-loop loses the run's entire progress; nothing here waits — an `ApprovalStep` would need to literally block this thread for however long a human takes to respond; and nothing here retries a flaky connector call. Every one of §21 through §53 is a specific, necessary answer to one of those gaps — none of them invalidates this loop's basic shape (a ready-queue driven walk over a graph), they make it durable, non-blocking, and resilient.

---

# 21. Follow-up Question 5 — "What Happens If the Worker Process Crashes Mid-Run?"

> **Interviewer:** *"Say step 3 of a 10-step workflow just finished, and the process running §20's loop crashes right before moving to step 4. What happens to this run?"*

With §20's loop as written: **the run is gone.** `context`, `completed`, and `ready` are all local, in-memory state — the moment the process dies, nothing durable remembers that steps 1-3 already ran, and restarting from scratch would re-execute them, very possibly re-sending an email or re-charging a customer a second time. This is the single most important reliability property a real workflow engine has to solve, and §22-§23 solve it the same way every durable-execution system (Temporal, Cadence, AWS Step Functions) actually does: **never treat in-memory state as the source of truth — treat a durable, append-only log as the source of truth, and derive everything else from it.**

---

# 22. Durable Execution: An Event Log Per Run

```java
public sealed interface WorkflowEvent permits RunStarted, StepStarted, StepSucceeded, StepFailed, RunCompleted, RunFailed {
    String runId();
    Instant occurredAt();
}

public record RunStarted(String runId, Instant occurredAt, String workflowId, int definitionVersion, Map<String, Object> triggerInput) implements WorkflowEvent { }
public record StepStarted(String runId, Instant occurredAt, String stepId, int attempt, Map<String, Object> input) implements WorkflowEvent { }
public record StepSucceeded(String runId, Instant occurredAt, String stepId, int attempt, Map<String, Object> output) implements WorkflowEvent { }
public record StepFailed(String runId, Instant occurredAt, String stepId, int attempt, String errorMessage) implements WorkflowEvent { }
public record RunCompleted(String runId, Instant occurredAt, Map<String, Object> finalOutput) implements WorkflowEvent { }
public record RunFailed(String runId, Instant occurredAt, String reason) implements WorkflowEvent { }
```

```java
public interface EventLog {
    void append(WorkflowEvent event);              // durable BEFORE the caller is told it succeeded -- the same
                                                     // WAL discipline the TinyDB Storage Engine guide builds (§13 there)
    List<WorkflowEvent> readAll(String runId);      // in append order -- the entire history of one run
}
```

Every fact about a run — which steps have started, succeeded, failed, and how many times each was attempted — is captured as one of these immutable events, appended in order, and **never mutated after the fact.** This is the Event Sourcing pattern the [Real-Time Collaboration guide](<Design a Real-Time Collaboration Tool — HLD, LLD, and Class Design From Scratch.md>) already builds for a different reason (concurrent-edit convergence); here it exists for durability: a `WorkflowRun`'s current status, and a `StepExecution`'s current status, are never stored as the primary record — they are **derived** by replaying `readAll(runId)` forward, which is exactly what makes crash recovery (§23) and a multi-day approval resume (§53) both reducible to "replay the log, then continue."

---

# 23. Resuming a Run After a Crash: Replay and Idempotent Re-Dispatch

```java
public final class OrchestrationEngine {

    private final EventLog eventLog;
    private final ActionExecutorRegistry registry;
    private final WorkflowDefinitionRegistry definitions; // §58

    public OrchestrationEngine(EventLog eventLog, ActionExecutorRegistry registry, WorkflowDefinitionRegistry definitions) {
        this.eventLog = eventLog;
        this.registry = registry;
        this.definitions = definitions;
    }

    /** Called once per run, on startup -- rebuilds in-memory state purely from durable events, then keeps going. */
    public void resume(String runId) {
        List<WorkflowEvent> history = eventLog.readAll(runId);
        if (history.isEmpty()) return; // not a real run, or already fully replayed elsewhere

        RunStarted started = (RunStarted) history.get(0);
        WorkflowDefinition definition = definitions.resolve(started.workflowId(), started.definitionVersion()); // §58 -- pinned version

        Map<String, Object> context = new HashMap<>();
        context.put("trigger", started.triggerInput());
        Set<String> succeededStepIds = new HashSet<>();

        for (WorkflowEvent event : history) {
            switch (event) {
                case StepSucceeded succeeded -> {
                    context.put(succeeded.stepId(), succeeded.output());
                    succeededStepIds.add(succeeded.stepId());
                }
                case RunCompleted ignored -> { return; } // already finished before the crash -- nothing to resume
                case RunFailed ignored -> { return; }
                default -> { /* RunStarted/StepStarted/StepFailed carry no state to rebuild here */ }
            }
        }

        continueFrom(definition, context, succeededStepIds, runId);
    }

    private void continueFrom(WorkflowDefinition definition, Map<String, Object> context, Set<String> succeeded, String runId) {
        for (String stepId : readyStepsGiven(definition, succeeded)) {
            dispatch(definition.stepsById().get(stepId), context, runId); // enqueued to the worker pool, §39
        }
    }

    // readyStepsGiven(...) is §44-§46's graph-walking logic (branching, joins, and loops); dispatch(...) is elided here, built in full in §39.
}
```

The idempotency this depends on: a step whose `StepSucceeded` event is already in the log is **never re-executed** — `succeededStepIds` short-circuits it, exactly the same "does this already reflect the change?" idempotency check the [TinyDB Storage Engine guide's redo phase](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) uses `page.pageLsn() >= record.lsn()` for. A step that only got as far as `StepStarted` before the crash (no matching `StepSucceeded` yet) is correctly treated as *not yet done*, and re-dispatched — which is exactly why every `ActionExecutor` (§37) must be safe to call more than once for the same logical attempt, using the idempotency keys §50 builds.

---

# 24. Follow-up Question 6 — "How Do Triggers Actually Kick Off a Run — Webhook, Schedule, Event?"

> **Interviewer:** *"You keep saying 'a run starts.' Where does that first `RunStarted` event actually come from?"*

Four distinct trigger types, each with a genuinely different mechanism behind it — §25 names all four; §26-§27 build the two with real scheduling/ingestion logic worth showing in code.

---

# 25. Trigger Types: Webhook, Schedule, Internal Event, Manual

- **Webhook**: an external system POSTs to `/webhooks/{triggerId}` (§14) the moment something happens on its end (a form submission, a payment). §27 builds ingestion, signature verification, and de-duplication.
- **Schedule (cron)**: a recurring, time-based fire — "every day at 9am," "every 15 minutes." §26 builds the scheduler.
- **Internal event**: MiniFlow's own event bus (an issue changed, a user signed up) triggers a workflow — structurally identical to [the JIRA-style guide's Observer-pattern event system](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>), just with "start a workflow run" as the listener's action instead of "send a notification."
- **Manual**: a user or API caller explicitly starts a run via `POST /workflows/{id}/runs` (§14) — no scheduling or ingestion machinery needed at all.

---

# 26. The Schedule Trigger: A Time-Ordered Priority Queue of Next-Fire-Times

```java
public record ScheduledTrigger(String triggerId, String workflowId, String cronExpression, Instant nextFireTime) { }

public final class TriggerScheduler {

    private final PriorityQueue<ScheduledTrigger> heap =
            new PriorityQueue<>(Comparator.comparing(ScheduledTrigger::nextFireTime));
    private final CronExpressionEvaluator cronEvaluator; // computes the next fire time after a given instant
    private final Consumer<ScheduledTrigger> onFire;      // starts a run, §23's entry point

    public TriggerScheduler(CronExpressionEvaluator cronEvaluator, Consumer<ScheduledTrigger> onFire) {
        this.cronEvaluator = cronEvaluator;
        this.onFire = onFire;
    }

    public synchronized void register(String triggerId, String workflowId, String cronExpression) {
        Instant next = cronEvaluator.nextFireTimeAfter(cronExpression, Instant.now());
        heap.add(new ScheduledTrigger(triggerId, workflowId, cronExpression, next));
    }

    /** Called once per polling tick by a single background thread. */
    public synchronized void tick() {
        Instant now = Instant.now();
        while (!heap.isEmpty() && !heap.peek().nextFireTime().isAfter(now)) {
            ScheduledTrigger due = heap.poll();
            onFire.accept(due);
            Instant next = cronEvaluator.nextFireTimeAfter(due.cronExpression(), now);
            heap.add(new ScheduledTrigger(due.triggerId(), due.workflowId(), due.cronExpression(), next));
        }
    }
}
```

A min-heap ordered by `nextFireTime` means `tick()` never has to scan every registered trigger to find which ones are due — it only ever looks at the single soonest one, `heap.peek()`, and stops the moment that one isn't due yet either. Re-inserting each trigger with its *newly computed* next fire time, immediately after it fires, is what keeps a recurring cron trigger recurring without needing a separate "recompute all schedules" pass.

---

# 27. The Webhook Trigger: Ingestion, Verification, and De-duplication

A webhook handler has three jobs, in this order, before it's allowed to start a run: **verify** the request is genuinely from the expected source (an HMAC signature check against a per-trigger shared secret — never trust an unauthenticated POST to kick off a tenant's workflow); **de-duplicate** (many webhook providers retry a delivery on a timeout even if the first attempt actually succeeded — a `providerDeliveryId`, stored and checked against previously-seen IDs for this trigger, turns "maybe delivered twice" into "processed exactly once"); then **start the run** by appending a `RunStarted` event (§22) with the webhook's payload as the trigger input. Skipping de-duplication specifically is a common, expensive mistake (§67) — a payment-provider webhook retried three times without it means three workflow runs, and very possibly three emails or three shipments, for one real event.

---

# 28. High-Level Architecture Diagram, Assembled

```text
                              Webhook            Cron Scheduler         Internal Event Bus
                            (§27, HMAC-           (§26, min-heap          (Observer pattern,
                          verified, de-duped)      of next-fire)           JIRA-guide style)
                                  \                    |                       /
                                   \                   |                      /
                                    v                   v                    v
                                          Trigger Service --> append RunStarted (§22)
                                                          |
                                                          v
                                          Orchestration Engine (§23, §44-§46)
                                     reads the Event Log, decides what's ready
                                          /              |               \
                                         v                v                v
                                Worker Pool         Worker Pool       Worker Pool
                               (HTTP conn., §39)   (email conn.)     (custom code,
                                                                      sandboxed)
                                         \                |                /
                                          v                v               v
                                            External systems / connectors
                                                          |
                                                          v
                                       StepStarted / StepSucceeded / StepFailed
                                              appended back to the Event Log
                                                          |
                                                          v
                                            Definition Store (versioned, §58)
                                            resolves which DAG this run follows
```

Every arrow in this diagram is a call this guide has already built real code for — nothing here is drawn from memory of what a workflow engine "probably" looks like.

---

# 29. Follow-up Question 7 — "Let's Get Concrete. Design the Class Model for a Run and Its Steps."

> **Interviewer:** *"You've shown me the definition side. Now show me what actually exists, as data, while a specific run is executing."*

§30 builds `WorkflowRun` and `StepExecution` as real classes; §31 assembles the full domain-entity class diagram, definition side and runtime side together.

---

# 30. The WorkflowRun and StepExecution Classes

```java
public final class WorkflowRun {

    private final String runId;
    private final String workflowId;
    private final int definitionVersion;      // pinned at start -- §58's sticky-versioning guarantee
    private final Instant startedAt;
    private RunStatus status;                 // §34's State pattern -- never a raw enum switch
    private final Map<String, StepExecution> stepExecutionsByStepId = new LinkedHashMap<>();

    public WorkflowRun(String runId, String workflowId, int definitionVersion, Instant startedAt) {
        this.runId = runId;
        this.workflowId = workflowId;
        this.definitionVersion = definitionVersion;
        this.startedAt = startedAt;
        this.status = RunStatus.RUNNING;
    }

    public String runId() { return runId; }
    public int definitionVersion() { return definitionVersion; }
    public RunStatus status() { return status; }
    public void transitionTo(RunStatus next) { this.status = status.transitionTo(next); } // §34

    public StepExecution stepExecution(String stepId) { return stepExecutionsByStepId.get(stepId); }
    public void record(StepExecution execution) { stepExecutionsByStepId.put(execution.stepId(), execution); }
    public Collection<StepExecution> allStepExecutions() { return stepExecutionsByStepId.values(); }
}
```

```java
public final class StepExecution {

    private final String stepId;
    private int attempt;
    private StepStatus status;                // §33's State pattern
    private Map<String, Object> input;
    private Map<String, Object> output;
    private String lastErrorMessage;

    public StepExecution(String stepId) {
        this.stepId = stepId;
        this.attempt = 0;
        this.status = StepStatus.PENDING;
    }

    public String stepId() { return stepId; }
    public int attempt() { return attempt; }
    public StepStatus status() { return status; }
    public Map<String, Object> output() { return output; }

    public void startAttempt(Map<String, Object> input) {
        this.attempt++;
        this.input = input;
        this.status = status.transitionTo(StepStatus.RUNNING); // §33
    }
    public void succeed(Map<String, Object> output) {
        this.output = output;
        this.status = status.transitionTo(StepStatus.SUCCEEDED);
    }
    public void fail(String errorMessage) {
        this.lastErrorMessage = errorMessage;
        this.status = status.transitionTo(StepStatus.FAILED);
    }
}
```

Both classes intentionally never expose a raw setter for `status` — every transition goes through `transitionTo`, which (§33-§34) validates the transition is actually legal before allowing it, the same discipline [the JIRA-style guide's issue-status workflow](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already applies to a business-domain status field, here applied to the engine's own execution lifecycle instead.

---

# 31. Class Diagram: Core Domain Entities

```text
WorkflowDefinition                          WorkflowRun
+ workflowId, version                       + runId, workflowId, definitionVersion
+ triggerType                               + status: RunStatus
+ stepsById: Map<String,StepDefinition>     + stepExecutionsByStepId
+ startStepId                                     |
      | 1                                          | 1..*
      | contains                                   | tracks
      v *                                          v
StepDefinition (sealed)                     StepExecution
+ ActionStep                                + stepId, attempt
+ ConditionStep                             + status: StepStatus
+ ParallelStep / JoinStep                    + input, output, lastErrorMessage
+ ForEachStep
+ ApprovalStep
+ SubWorkflowStep
      | *
      | connects via
      v
    Edge
+ fromStepId, toStepId, condition


Trigger (bound to one WorkflowDefinition)          WorkflowEvent (sealed, §22)
+ ScheduleTrigger (cron)                            + RunStarted / StepStarted
+ WebhookTrigger (HMAC-verified)                    + StepSucceeded / StepFailed
+ EventTrigger (internal bus)                       + RunCompleted / RunFailed
                                                           |
                                                           | replayed to derive
                                                           v
                                                  WorkflowRun + StepExecution
                                                  (never the primary record, §22)
```

The two halves of this diagram deliberately never merge: the **definition side** (top-left) is static, versioned data describing what *could* happen; the **runtime side** (top-right, bottom-right) is what *actually* happened, one run at a time, always derivable from the event log alone. Conflating them — storing a run's live status as a mutable field directly on a shared, cached `WorkflowDefinition` object, say — is exactly the kind of shortcut that breaks the crash-recovery story §23 depends on.

---

# 32. Follow-up Question 8 — "How Do You Model Run/Step Status So the Logic Isn't a Giant If/Else?"

> **Interviewer:** *"A step can be pending, running, succeeded, failed, retrying, waiting on a human, or skipped. A run can be running, completed, failed, or cancelled. How do you keep the rules for 'what can transition to what' from turning into an unmaintainable tangle of conditionals?"*

The exact same answer [the JIRA-style guide gives for issue status](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>), applied to a different status field: the **State pattern**. Each status is an object that knows its own legal next-states, so "is this transition allowed" is a single polymorphic call, never a hand-maintained table of `if (current == X && next == Y)` checks scattered across the codebase.

---

# 33. The State Pattern for Step Execution Status

```java
public enum StepStatus {
    PENDING {
        @Override public StepStatus transitionTo(StepStatus next) {
            return requireLegal(next, Set.of(RUNNING, SKIPPED));
        }
    },
    RUNNING {
        @Override public StepStatus transitionTo(StepStatus next) {
            return requireLegal(next, Set.of(SUCCEEDED, FAILED, WAITING));
        }
    },
    WAITING {                                     // §52-§53 -- a human approval step, possibly for days
        @Override public StepStatus transitionTo(StepStatus next) {
            return requireLegal(next, Set.of(SUCCEEDED, FAILED));
        }
    },
    FAILED {
        @Override public StepStatus transitionTo(StepStatus next) {
            return requireLegal(next, Set.of(RUNNING)); // a retry -- §49
        }
    },
    SUCCEEDED {
        @Override public StepStatus transitionTo(StepStatus next) {
            throw new IllegalStateException("SUCCEEDED is terminal -- cannot transition to " + next);
        }
    },
    SKIPPED {
        @Override public StepStatus transitionTo(StepStatus next) {
            throw new IllegalStateException("SKIPPED is terminal -- cannot transition to " + next);
        }
    };

    public abstract StepStatus transitionTo(StepStatus next);

    protected StepStatus requireLegal(StepStatus next, Set<StepStatus> legalNextStates) {
        if (!legalNextStates.contains(next)) {
            throw new IllegalStateException("Illegal step transition: " + this + " -> " + next);
        }
        return next;
    }
}
```

`FAILED -> RUNNING` being legal, but nothing else being able to reach `RUNNING` a second time except through `FAILED`, is what makes §49's retry logic *structurally* correct — a step can only ever be re-attempted after actually failing, never accidentally re-run from `SUCCEEDED` or mid-flight from `WAITING`.

---

# 34. The State Pattern for Run-Level Status

```java
public enum RunStatus {
    RUNNING {
        @Override public RunStatus transitionTo(RunStatus next) {
            return requireLegal(next, Set.of(COMPLETED, FAILED, CANCELLED));
        }
    },
    COMPLETED {
        @Override public RunStatus transitionTo(RunStatus next) {
            throw new IllegalStateException("COMPLETED is terminal -- cannot transition to " + next);
        }
    },
    FAILED {
        @Override public RunStatus transitionTo(RunStatus next) {
            throw new IllegalStateException("FAILED is terminal -- cannot transition to " + next);
        }
    },
    CANCELLED {
        @Override public RunStatus transitionTo(RunStatus next) {
            throw new IllegalStateException("CANCELLED is terminal -- cannot transition to " + next);
        }
    };

    public abstract RunStatus transitionTo(RunStatus next);

    protected RunStatus requireLegal(RunStatus next, Set<RunStatus> legalNextStates) {
        if (!legalNextStates.contains(next)) {
            throw new IllegalStateException("Illegal run transition: " + this + " -> " + next);
        }
        return next;
    }
}
```

A run's status has exactly one non-terminal state (`RUNNING`) and three terminal ones — deliberately simpler than a step's, because a run's overall outcome is a *summary* of its steps' outcomes (a run is `COMPLETED` once the graph walker, §44-§46, finds no further ready steps and every reachable step has reached a terminal `StepStatus`), not an independent thing with its own rich lifecycle.

---

# 35. Class Diagram: The Execution State Machine

```text
                    StepStatus                                RunStatus
                                                                  RUNNING
                    PENDING                                     /   |   \
                   /       \                                   v    v    v
                  v         v                              COMPLETED FAILED CANCELLED
              RUNNING    SKIPPED (terminal)               (terminal) (terminal) (terminal)
             /   |   \
            v    v    v
      SUCCEEDED FAILED WAITING
     (terminal)  |        |
                 v        v
               RUNNING  SUCCEEDED / FAILED
              (retry, §49)
```

Every arrow in this diagram corresponds to exactly one `Set.of(...)` entry in §33/§34's code — the diagram and the code are two views of the identical rule set, which is the entire point of the State pattern: the legal-transitions table lives in exactly one place, not duplicated between a diagram nobody re-checks and code that quietly drifts from it.

---

# 36. Follow-up Question 9 — "How Do Users Add a New Integration Without Touching Your Core Engine?"

> **Interviewer:** *"Today you support HTTP and email. Tomorrow someone wants Slack, then Stripe, then a custom Python script. How does adding the hundredth connector not mean editing the orchestration engine's source for the hundredth time?"*

The Strategy pattern, plus a registry keyed by connector type — the exact mechanism that makes `ActionStep.connectorType()` (§17) a lookup key instead of a hard-coded branch. §37 defines the interface and registry; §38 implements two real connectors against it; §39 explains why each connector type gets its own worker pool, not a shared one.

---

# 37. The ActionExecutor Strategy Interface and a Plugin Registry

```java
public interface ActionExecutor {
    String connectorType();                                                  // e.g. "http", "email", "slack"
    Map<String, Object> execute(Map<String, Object> resolvedInput, IdempotencyKey idempotencyKey) throws ActionExecutionException;
}
```

```java
public final class ActionExecutorRegistry {

    private final Map<String, ActionExecutor> executorsByType = new ConcurrentHashMap<>(); // registered at startup, resolved by every worker thread concurrently

    public void register(ActionExecutor executor) {
        executorsByType.put(executor.connectorType(), executor);
    }

    public ActionExecutor resolve(String connectorType) {
        ActionExecutor executor = executorsByType.get(connectorType);
        if (executor == null) {
            throw new IllegalArgumentException("No ActionExecutor registered for connector type: " + connectorType);
        }
        return executor;
    }
}
```

Adding Slack support is now: implement `ActionExecutor`, call `register(new SlackActionExecutor(...))` once at startup — nothing in `OrchestrationEngine` (§23), `WorkflowValidator` (§18), or any other core class changes. This is the Open/Closed Principle made concrete (§66): the system is open to a new connector type, closed to modification of the code that dispatches to one.

---

# 38. Implementing Two Concrete ActionExecutors: HTTP and Email

```java
public final class HttpActionExecutor implements ActionExecutor {

    private final HttpClient httpClient;

    public HttpActionExecutor(HttpClient httpClient) { this.httpClient = httpClient; }

    @Override public String connectorType() { return "http"; }

    @Override
    public Map<String, Object> execute(Map<String, Object> input, IdempotencyKey idempotencyKey) throws ActionExecutionException {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create((String) input.get("url")))
                    .header("Idempotency-Key", idempotencyKey.value()) // §50 -- the callee's job to de-duplicate on this
                    .POST(HttpRequest.BodyPublishers.ofString((String) input.getOrDefault("body", "")))
                    .build();
            HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() >= 500) {
                throw new ActionExecutionException("HTTP " + response.statusCode() + " -- treated as retryable"); // §49
            }
            return Map.of("statusCode", response.statusCode(), "body", response.body());
        } catch (IOException | InterruptedException e) {
            throw new ActionExecutionException("HTTP call failed: " + e.getMessage(), e); // network errors -- retryable
        }
    }
}
```

```java
public final class EmailActionExecutor implements ActionExecutor {

    private final EmailClient emailClient;

    public EmailActionExecutor(EmailClient emailClient) { this.emailClient = emailClient; }

    @Override public String connectorType() { return "email"; }

    @Override
    public Map<String, Object> execute(Map<String, Object> input, IdempotencyKey idempotencyKey) throws ActionExecutionException {
        // Most email providers accept an idempotency/message key directly -- passing it through means a retried
        // send after a timeout, where the FIRST send actually succeeded, never results in a duplicate email.
        String messageId = emailClient.send((String) input.get("to"), (String) input.get("subject"),
                (String) input.get("body"), idempotencyKey.value());
        return Map.of("messageId", messageId);
    }
}
```

Both throw `ActionExecutionException` for anything §49 should retry, and return a plain `Map<String, Object>` output — the same shape `StepSucceeded.output()` (§22) stores and later steps' expression templates (§42) read from, deliberately, so a connector author never needs to think about the event log or the expression language at all.

---

# 39. Class Diagram: The Pluggable Action Layer, and Per-Connector Worker Pools

```text
      ActionExecutor  <<interface>>
      + connectorType(): String
      + execute(input, idempotencyKey): Map
             ^
             | implements
     +-------+---------+------------------+
     |                 |                  |
HttpActionExecutor  EmailActionExecutor  SlackActionExecutor (future, §67)


      ActionExecutorRegistry
      + register(executor)
      + resolve(connectorType): ActionExecutor
             |
             | one bounded worker pool PER connectorType, e.g.:
             v
   "http"   -> ThreadPoolExecutor(core=20, max=50, queue=500)
   "email"  -> ThreadPoolExecutor(core=5,  max=10, queue=1000)
   "custom" -> a sandboxed pool, isolated by design (§67)
```

A single shared pool for every connector type would let one slow, misbehaving integration (a partner's API having a bad day) starve dispatch for every *other* connector, including fast, healthy ones — the exact "isolate slow tasks from fast ones with separate pools" argument the [Executor Framework guide](<Build Your Own Executor Framework From Scratch — A Java Concurrency Step-by-Step Guide.md>) makes in general, applied here per connector type specifically because that's the natural failure-isolation boundary in this domain.

---

# 40. Follow-up Question 10 — "How Do You Pass One Step's Output Into the Next Step's Input?"

> **Interviewer:** *"Step 2 needs the customer email address that step 1's HTTP call returned. The user configures this in the workflow builder UI, not in Java code. How does that actually work at runtime?"*

An **expression language**, evaluated against the run's accumulated `context` map (§20, §23) — every step's declared `inputTemplate` (§17's `ActionStep`) contains placeholders like `{{steps.step1.output.email}}`, resolved just before that step executes. §41 designs the syntax; §42 implements the evaluator.

---

# 41. An Expression Language for Data Mapping Between Steps

The syntax deliberately stays small: a dotted path inside `{{ }}`, rooted at either `trigger` (the run's original input) or `steps.<stepId>.output` (a previously-executed step's result):

```text
{{trigger.customerId}}
{{steps.fetchCustomer.output.email}}
{{steps.checkInventory.output.items[0].sku}}
```

This is the same "small, purpose-built expression grammar, not a general-purpose scripting language" decision the [TinyDB Query Engine guide's `Expression`/`ExpressionEvaluator`](<TinyDB Query Engine and SQL Parser — Step-by-Step Implementation Guide.md>) makes for SQL `WHERE` clauses — embedding a full scripting language here would be a substantial security surface (arbitrary code referencing arbitrary context data) for a feature that only ever needs to *read* a value out of a nested map.

---

# 42. Implementing a Simple Expression Evaluator

```java
public final class ExpressionEvaluator {

    private static final Pattern PLACEHOLDER = Pattern.compile("\\{\\{\\s*([^}]+?)\\s*\\}\\}");

    public String resolveTemplate(String template, Map<String, Object> context) {
        Matcher matcher = PLACEHOLDER.matcher(template);
        StringBuilder result = new StringBuilder();
        int lastEnd = 0;
        while (matcher.find()) {
            result.append(template, lastEnd, matcher.start());
            Object resolved = resolvePath(matcher.group(1), context);
            result.append(resolved == null ? "" : resolved.toString());
            lastEnd = matcher.end();
        }
        result.append(template, lastEnd, template.length());
        return result.toString();
    }

    public Object resolvePath(String path, Map<String, Object> context) {
        Object current = context;
        for (String segment : path.split("\\.")) {
            String key = segment;
            Integer index = null;
            int bracket = segment.indexOf('[');
            if (bracket != -1) {
                key = segment.substring(0, bracket);
                index = Integer.parseInt(segment.substring(bracket + 1, segment.indexOf(']')));
            }
            if (!(current instanceof Map<?, ?> map)) return null;
            current = map.get(key);
            if (index != null) {
                if (!(current instanceof List<?> list) || index >= list.size()) return null;
                current = list.get(index);
            }
        }
        return current;
    }
}
```

Every step's `inputTemplate` (a `Map<String, String>`, §17) is resolved field-by-field through `resolveTemplate` immediately before dispatch — `ActionStep`'s input to `execute()` (§38) is always the *fully resolved* map, never a template string, which is precisely why an `ActionExecutor` implementation never needs to know this expression language exists at all: by the time it sees an input map, every placeholder is already gone.

---

# 43. Follow-up Question 11 — "How Do You Support Branching, Parallel Fan-Out/Fan-In, and Loops?"

> **Interviewer:** *"Real workflows aren't a straight line. Show me a conditional branch, two steps running in parallel that both feed into a third, and a loop over a list of items."*

§17 already defined the four step types this needs (`ConditionStep`, `ParallelStep`, `JoinStep`, `ForEachStep`); §44-§46 build the graph-walking logic that gives each one real, correct semantics.

---

# 44. Conditional Branches and the Graph Walker

A `ConditionStep`'s `expression` (§17) is evaluated with §42's evaluator against the run's context, and the **edge's own `condition`** (§17's `Edge`, e.g. `"true"`/`"false"`, or an arbitrary boolean expression) decides which outgoing edge(s) actually fire:

```java
private List<String> nextStepsAfter(StepDefinition step, Map<String, Object> context) {
    List<String> next = new ArrayList<>();
    for (Edge edge : step.outgoingEdges()) {
        if (edge.condition() == null || evaluateBoolean(edge.condition(), context)) {
            next.add(edge.toStepId());
        }
    }
    return next;
}
```

A `ConditionStep` is deliberately *not* special-cased with its own dispatch branch — it's just a step whose outgoing edges happen to carry conditions, and the exact same `nextStepsAfter` walk that every other step type already uses handles it for free. This is why `ConditionStep`'s `sealed interface` case (§17) has no `expression`-specific execution logic of its own beyond contributing to which edges are considered — the branching *is* the edge conditions, not a separate mechanism.

---

# 45. Parallel Fan-Out and Fan-In: Join Semantics

A `ParallelStep` (§17) has multiple outgoing edges with no conditions at all — `nextStepsAfter` (§44) already fires every one of them, so fan-out needs no new mechanism. Fan-**in** is the genuinely new piece: a `JoinStep` must wait for **all** of its expected incoming branches before it's allowed to run, not just the first one to finish:

```java
public boolean isReady(JoinStep join, Set<String> succeededStepIds, WorkflowDefinition definition) {
    long incomingSucceeded = definition.stepsById().values().stream()
            .flatMap(s -> s.outgoingEdges().stream())
            .filter(edge -> edge.toStepId().equals(join.id()))
            .map(Edge::fromStepId)
            .filter(succeededStepIds::contains)
            .count();
    return incomingSucceeded >= join.expectedIncomingCount(); // §17 -- declared explicitly, not inferred
}
```

Declaring `expectedIncomingCount` explicitly on the `JoinStep` (§17), rather than inferring it by counting incoming edges in the graph, matters for one specific case: a `ParallelStep` whose branches include a `ConditionStep` might legitimately have fewer than all branches actually complete (one branch's condition was false, so it never even started) — an explicit expected count, set correctly at design time, is what lets a join distinguish "waiting on a branch that's still running" from "this branch was never going to run at all," which a pure edge-count would get wrong.

---

# 46. The For-Each Loop Step: Iterating Over a Collection

```java
public record ForEachStepExecution(String stepId, List<Object> items, int nextIndex) { }

private List<Map<String, Object>> executeForEach(ForEachStep step, Map<String, Object> context) {
    Object rawCollection = expressionEvaluator.resolvePath(step.collectionExpression(), context); // §42
    List<?> items = (rawCollection instanceof List<?> list) ? list : List.of();

    List<Map<String, Object>> results = new ArrayList<>();
    for (Object item : items) {
        Map<String, Object> iterationContext = new HashMap<>(context);
        iterationContext.put("item", item); // §17's step.body() references {{item}}, resolved per iteration
        results.add(executeStep(step.body(), iterationContext)); // recursive -- body is itself a StepDefinition
    }
    return results;
}
```

Making `ForEachStep.body()` (§17) itself a `StepDefinition`, rather than a special-cased list of actions, means the loop body can be *any* step type this engine supports — including another `ActionStep`, a nested `ConditionStep`, or even another `ForEachStep` — for free, purely because `executeStep` is written generically over the `sealed interface` rather than assuming what kind of step it's given.

---

# 47. Follow-up Question 12 — "External Calls Are Flaky. How Do You Retry Without Duplicating Side Effects?"

> **Interviewer:** *"A Slack call times out. Was the message actually sent, or not? If you retry blindly, what stops a user from getting the same Slack message three times?"*

Two genuinely separate concerns, each needing its own mechanism: **when and how often to retry** (§48-§49 — backoff), and **making a retried call safe even if the previous attempt actually succeeded despite looking like it failed** (§50 — idempotency keys). Confusing these two, or building only the first, is the single most common mistake in a homegrown workflow engine (§67).

---

# 48. Retry Policy: Exponential Backoff With Jitter

```java
public interface RetryPolicy {
    boolean shouldRetry(int attemptNumber, Exception lastError);
    Duration delayBefore(int attemptNumber);
}
```

```java
public final class ExponentialBackoffRetryPolicy implements RetryPolicy {

    private final int maxAttempts;
    private final Duration baseDelay;
    private final Duration maxDelay;

    public ExponentialBackoffRetryPolicy(int maxAttempts, Duration baseDelay, Duration maxDelay) {
        this.maxAttempts = maxAttempts;
        this.baseDelay = baseDelay;
        this.maxDelay = maxDelay;
    }

    @Override
    public boolean shouldRetry(int attemptNumber, Exception lastError) {
        if (attemptNumber >= maxAttempts) return false;
        return lastError instanceof ActionExecutionException; // §38 -- only errors explicitly marked retryable
    }

    @Override
    public Duration delayBefore(int attemptNumber) {
        long exponentialMillis = baseDelay.toMillis() * (1L << Math.min(attemptNumber, 20)); // cap the shift, avoid overflow
        long cappedMillis = Math.min(exponentialMillis, maxDelay.toMillis());
        long jitterMillis = ThreadLocalRandom.current().nextLong(cappedMillis / 2, cappedMillis + 1);
        return Duration.ofMillis(jitterMillis);
    }
}
```

**Jitter** — randomizing within a range rather than always waiting the exact computed delay — exists for a specific, well-documented reason: without it, a burst of steps that all failed at the same instant (a downstream outage) would all retry at exactly the same future instant too, turning one outage into a synchronized "thundering herd" the moment the downstream recovers. Randomizing spreads that retry burst out instead of concentrating it.

---

# 49. Applying the Retry Policy: Where It Plugs Into the Engine

```java
private void dispatchWithRetry(StepDefinition step, StepExecution execution, Map<String, Object> context, RetryPolicy retryPolicy) {
    try {
        execution.startAttempt(resolvedInput(step, context)); // §33's PENDING/FAILED -> RUNNING
        Map<String, Object> output = registry.resolve(connectorTypeOf(step))
                .execute(execution.input(), IdempotencyKey.forAttempt(execution)); // §50
        execution.succeed(output); // §33's RUNNING -> SUCCEEDED
        eventLog.append(new StepSucceeded(runId, Instant.now(), step.id(), execution.attempt(), output));
    } catch (ActionExecutionException e) {
        execution.fail(e.getMessage()); // §33's RUNNING -> FAILED
        eventLog.append(new StepFailed(runId, Instant.now(), step.id(), execution.attempt(), e.getMessage()));
        if (retryPolicy.shouldRetry(execution.attempt(), e)) {
            scheduleRetryAfter(retryPolicy.delayBefore(execution.attempt()), () -> dispatchWithRetry(step, execution, context, retryPolicy));
        } else {
            sendToDeadLetterQueue(step, execution, runId); // §51 -- permanently failed, needs a human
        }
    }
}
```

Every retry attempt is its own `StepStarted`/`StepFailed` pair in the event log (§22), each carrying its own `attempt` number — a run's full history shows exactly how many times a step was tried and why each attempt failed, which is precisely the debugging visibility §7's "run history" requirement asks for.

---

# 50. Idempotency Keys for Side-Effecting Steps

```java
public record IdempotencyKey(String value) {
    public static IdempotencyKey forAttempt(StepExecution execution) {
        // Deterministic and STABLE across a re-dispatch of the SAME logical attempt (e.g. after a crash and
        // resume, §23) -- runId+stepId+attempt is unique per real-world side effect this engine intends to
        // cause, and identical every time that exact attempt is (re-)dispatched, which is exactly what lets
        // a downstream connector (Stripe, an email provider, this guide's own §38 executors) recognize "I've
        // already done this" and return the ORIGINAL result instead of performing the action a second time.
        return new IdempotencyKey(execution.runId() + ":" + execution.stepId() + ":" + execution.attempt());
    }
}
```

This is the honest answer to §6's "exactly-once execution" question: MiniFlow itself only ever guarantees **at-least-once dispatch** of a step (a crash can always cause a re-dispatch, §23) — true exactly-once execution of an arbitrary external side effect isn't something an orchestrator can unilaterally guarantee, because it doesn't control the downstream system. What it *can* guarantee, and does, is that every dispatch of the same logical attempt carries the same idempotency key, so any downstream connector that honors it (as §38's `HttpActionExecutor`/`EmailActionExecutor` both do, passing it straight through) turns "retried at-least-once" into "effectively exactly-once" from the user's point of view — the same distinction a payment provider's own idempotency-key API is built around.

---

# 51. The Dead-Letter Queue for Permanently Failed Steps

A step that exhausts `RetryPolicy.shouldRetry` (§48) is not silently dropped — its `StepExecution` (still `FAILED`, §33) and its run are routed to a **dead-letter queue**: a durable, queryable list of "this needs a human" cases, surfaced in the run-history UI (§7) with the full failure history already captured in the event log (§22, every failed attempt's error message). A human can then either fix the underlying issue and trigger a manual retry (re-entering `dispatchWithRetry`, §49, with a fresh attempt count), or explicitly mark the run as permanently failed, which is exactly the trigger for §55's compensation logic if earlier steps in that same run already had real side effects.

---

# 52. Follow-up Question 13 — "An Approval Step Might Wait Three Days. How Do You Not Just Block a Thread?"

> **Interviewer:** *"A human needs to click 'approve' before the workflow continues, and that might take five minutes or five days. Your worker pools are bounded — what happens to the thread that's 'waiting' for that approval?"*

Nothing waits. §33 already gave `StepStatus` a `WAITING` state for exactly this; §53 shows precisely how a step *enters* `WAITING` (and gives up its thread entirely) and how an external callback, arriving whenever it arrives, *resumes* the run without anything having stayed blocked in the meantime.

---

# 53. Suspend and Resume: The WAITING State and External Callbacks

```java
private void dispatchApprovalStep(ApprovalStep step, StepExecution execution, String runId) {
    execution.startAttempt(Map.of());
    execution.transitionToWaiting(); // §33 -- RUNNING -> WAITING; the thread returns immediately, holds nothing
    eventLog.append(new StepStarted(runId, Instant.now(), step.id(), execution.attempt(), Map.of()));
    notifyApprovers(step.approverGroup(), runId, step.id()); // an email/Slack message with a link to §14's resume endpoint
    // No thread, connection, or lock is held past this point. The run is durably parked at WAITING
    // in the event log -- exactly the same "state lives in the log, not in memory" property §22 built
    // for crash recovery, now reused for a completely different reason: a wait that can outlast any
    // single process's uptime by orders of magnitude.
}

/** Called by POST /runs/{runId}/steps/{stepId}/resume (§14), whenever a human actually responds. */
public void resumeApprovalStep(String runId, String stepId, Map<String, Object> approvalOutcome) {
    eventLog.append(new StepSucceeded(runId, Instant.now(), stepId, currentAttempt(runId, stepId), approvalOutcome));
    resume(runId); // §23 -- the IDENTICAL replay-and-continue call crash recovery already uses
}
```

The reuse in that last line is the entire design insight this section exists to make explicit: **resuming after a crash and resuming after a three-day human approval are the same operation** — both are "append the fact that just became known, then replay the log and continue from wherever it now says to continue." Building `resume(runId)` once, in §23, and calling it from two different triggers (a process restart, an HTTP callback) is a direct, deliberate consequence of never treating in-memory state as authoritative in the first place.

---

# 54. Follow-up Question 14 — "Step 3 Failed After Steps 1 and 2 Already Had Real Side Effects. Now What?"

> **Interviewer:** *"Step 1 charged a customer's card. Step 2 created a shipment. Step 3 — reserving warehouse inventory — fails permanently. You can't just mark the run 'failed' and walk away; there's a charge and a shipment that shouldn't exist anymore. How do you design for that?"*

This is the hardest, most consequential follow-up in the entire workflow-automation domain, and the honest answer is: **you cannot undo an external side effect by rolling back a database transaction, because the side effect already happened outside your database.** The standard technique is the **Saga pattern** (§55) — every step that has a real-world effect also declares how to *compensate* for it, and a permanently-failed run walks its already-succeeded steps **in reverse**, calling each one's compensation.

---

# 55. The Saga Pattern: Compensating Actions

```java
public interface CompensableStep {
    Map<String, Object> compensate(Map<String, Object> originalOutput, Map<String, Object> originalInput) throws ActionExecutionException;
}

// An ActionExecutor MAY additionally implement CompensableStep -- not every action has something
// meaningful to compensate (an idempotent read has nothing to undo), but a charge or a shipment does.
public final class ChargeCardActionExecutor implements ActionExecutor, CompensableStep {

    private final PaymentGatewayClient gateway;

    public ChargeCardActionExecutor(PaymentGatewayClient gateway) { this.gateway = gateway; }

    @Override public String connectorType() { return "charge-card"; }

    @Override
    public Map<String, Object> execute(Map<String, Object> input, IdempotencyKey idempotencyKey) throws ActionExecutionException {
        String chargeId = gateway.charge((String) input.get("customerId"), (long) input.get("amountCents"), idempotencyKey.value());
        return Map.of("chargeId", chargeId);
    }

    @Override
    public Map<String, Object> compensate(Map<String, Object> originalOutput, Map<String, Object> originalInput) throws ActionExecutionException {
        gateway.refund((String) originalOutput.get("chargeId")); // the natural inverse of "charge"
        return Map.of("refunded", true);
    }
}
```

```java
private void runCompensation(WorkflowRun run, WorkflowDefinition definition) {
    List<StepExecution> succeededInOrder = run.allStepExecutions().stream()
            .filter(e -> e.status() == StepStatus.SUCCEEDED)
            .toList(); // already in execution order, §30's LinkedHashMap iteration order

    for (int i = succeededInOrder.size() - 1; i >= 0; i--) { // REVERSE order -- undo the most recent effect first
        StepExecution execution = succeededInOrder.get(i);
        StepDefinition step = definition.stepsById().get(execution.stepId());
        ActionExecutor executor = registry.resolve(connectorTypeOf(step));
        if (executor instanceof CompensableStep compensable) {
            try {
                Map<String, Object> compensationResult = compensable.compensate(execution.output(), execution.input());
                eventLog.append(new StepSucceeded(run.runId(), Instant.now(), execution.stepId() + ":compensation", 1, compensationResult));
            } catch (ActionExecutionException e) {
                // compensation itself failing is its own dead-letter case (§51) -- surfaced to a human, never swallowed
                eventLog.append(new StepFailed(run.runId(), Instant.now(), execution.stepId() + ":compensation", 1, e.getMessage()));
            }
        }
    }
}
```

Walking already-completed work **in reverse order**, undoing the most recent effect first, is precisely the same shape as the [TinyDB Storage Engine guide's Undo phase](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) walking a transaction's WAL records backward — a database transaction and a Saga are solving the same underlying problem (atomicity across a sequence of effects) at two different layers: one where the "effects" are page mutations with a beforeImage to restore to, the other where the effects are external systems with a domain-specific inverse operation (refund undoes charge; cancel-shipment undoes create-shipment) instead of a byte-level undo.

---

# 56. Implementing a Compensating Step's Trigger Point

Compensation is invoked from exactly one place: the moment a run's status is about to become `FAILED` (§34) after a step permanently exhausts retries and lands in the dead-letter queue (§51) — never automatically for a run a human hasn't yet decided is unrecoverable, since some dead-lettered runs *are* fixed and manually retried forward (§51) rather than rolled back. Whether compensation runs automatically or requires an explicit human "roll this back" action is itself a product decision worth stating explicitly in an interview — automatic compensation is more convenient but riskier (a compensation bug now runs unattended on every permanent failure), and a reasonable, defensible default is: automatic for steps explicitly marked safe to auto-compensate, human-gated for everything else.

---

# 57. Follow-up Question 15 — "A Definition Changes While Thousands of Runs Are Still In Flight. What Happens?"

> **Interviewer:** *"A user edits their workflow — adds a step, removes another — and publishes a new version. Ten thousand runs started under the OLD version are still executing. Do they suddenly see the new graph mid-run?"*

They must not — and the reason is exactly what §22-§23 already built: a run's correctness depends on replaying its event log against the *same* definition it started with. §58 makes the pinning this requires explicit.

---

# 58. Workflow Versioning: Sticky Versions for In-Flight Runs

```java
public final class WorkflowDefinitionRegistry {

    private final Map<String, NavigableMap<Integer, WorkflowDefinition>> versionsByWorkflowId = new ConcurrentHashMap<>();

    public void publish(WorkflowDefinition definition) {
        versionsByWorkflowId
                .computeIfAbsent(definition.workflowId(), id -> new ConcurrentSkipListMap<>())
                .put(definition.version(), definition);
    }

    /** The ONLY lookup a running WorkflowRun is ever allowed to use -- always an EXACT, pinned version. */
    public WorkflowDefinition resolve(String workflowId, int version) {
        WorkflowDefinition definition = versionsByWorkflowId.getOrDefault(workflowId, Collections.emptyNavigableMap()).get(version);
        if (definition == null) {
            throw new IllegalStateException("Workflow " + workflowId + " version " + version + " not found -- versions are never deleted, only superseded");
        }
        return definition;
    }

    /** The lookup a NEW trigger fire uses -- always the latest published version. */
    public WorkflowDefinition resolveLatest(String workflowId) {
        var versions = versionsByWorkflowId.get(workflowId);
        if (versions == null || versions.isEmpty()) throw new IllegalStateException("No published version for " + workflowId);
        return versions.lastEntry().getValue();
    }
}
```

`WorkflowRun.definitionVersion()` (§30) is set exactly once, from `RunStarted.definitionVersion()` (§22), at the moment a run begins — and `resume()` (§23) always calls `resolve(workflowId, started.definitionVersion())`, the pinned-exact-version lookup, never `resolveLatest`. A version is **never deleted, only superseded** by a newer one, precisely so that a run pinned to version 3 can still resolve it correctly on day 40, long after version 7 has become the default for brand-new triggers. This is the identical "old data must remain readable after the reading code evolves" discipline the [TinyDB Storage Engine guide's page header format](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) states for its own on-disk contract, applied here to a workflow's DAG shape instead of a page's byte layout.

---

# 59. Follow-up Question 16 — "You're At Millions of Runs a Day, Thousands of Tenants. What Breaks First?"

> **Interviewer:** *"Single orchestrator process, single event log, single set of worker pools. Where does this fall over first as load grows, and what's the standard fix?"*

Three distinct bottlenecks, each with its own standard answer: the **orchestration engine itself** running out of capacity to walk every in-flight run's graph (§60 — partition by run ID); **one tenant's traffic burst starving every other tenant** sharing the same connector worker pool (§61 — per-tenant rate limiting and backpressure); and **worker pool sizing** not adapting to load at all (§62 — queue-depth-driven autoscaling).

---

# 60. Partitioning the Execution Engine by Tenant and Run ID

```java
public final class RunPartitioner {

    private final int partitionCount;

    public RunPartitioner(int partitionCount) { this.partitionCount = partitionCount; }

    public int partitionFor(String runId) {
        // Consistent hashing, the same technique the File Storage guide's object-store design uses for
        // sharding objects across nodes -- here sharding RUNS across orchestrator instances instead.
        return Math.floorMod(runId.hashCode(), partitionCount);
    }
}
```

Every orchestrator instance is responsible for exactly the runs whose `partitionFor(runId)` matches its own assigned partition — a run's entire event log, and every event ever appended to it, only ever needs to be read by *one* orchestrator instance at a time, which is what makes horizontal scaling of the orchestration tier as simple as adding more instances and rebalancing partitions, the identical scaling story [the File Storage guide's consistent-hashing section](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) already tells for object storage nodes.

---

# 61. Backpressure and Rate Limiting Per Connector, Per Tenant

A shared connector worker pool (§39) protects against one connector *type* starving another, but not against one **tenant** starving every other tenant on the *same* connector — a tenant blasting ten thousand Slack messages a minute would otherwise consume the entire "slack" pool's capacity. The fix is a per-`(tenantId, connectorType)` token-bucket rate limiter checked *before* a step is even handed to the worker pool:

```java
public final class TokenBucketRateLimiter {

    private final Map<String, AtomicLong> tokensByKey = new ConcurrentHashMap<>();
    private final long capacity;
    private final long refillPerSecond;

    public TokenBucketRateLimiter(long capacity, long refillPerSecond) {
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
    }

    public boolean tryAcquire(String tenantId, String connectorType) {
        String key = tenantId + ":" + connectorType;
        AtomicLong tokens = tokensByKey.computeIfAbsent(key, k -> new AtomicLong(capacity));
        long current = tokens.get();
        if (current <= 0) return false;
        return tokens.compareAndSet(current, current - 1); // CAS, not a plain decrement -- the same check-then-act
                                                             // race the ConcurrentHashMap guide's bucket locking avoids,
                                                             // solved here with a lock-free retry instead of a lock
    }

    // A background refill task adds refillPerSecond tokens per key, once per second, capped at `capacity`.
}
```

A step whose `tryAcquire` returns `false` is not failed — it's simply **re-queued with a short delay** (reusing §49's own scheduled-retry mechanism), which is what turns "one noisy tenant" into "one noisy tenant's steps queue up a little longer," never "every tenant's steps queue up."

---

# 62. Worker Autoscaling and Queue-Depth-Driven Scheduling

Each connector's worker pool (§39) exposes its current queue depth as a metric; an autoscaler watches that metric per connector type and adjusts `corePoolSize` within the pool's configured `[min, max]` bounds — a sustained deep queue on the "http" pool scales *that* pool up independently of the "email" pool sitting comfortably idle. This is the same core-vs-max distinction the [Executor Framework guide](<Build Your Own Executor Framework From Scratch — A Java Concurrency Step-by-Step Guide.md>) builds from scratch, driven here by an external control loop instead of `ThreadPoolExecutor`'s own built-in queue-then-grow-to-max behavior, because a workflow engine additionally wants that scaling decision visible and tunable per connector, not just automatic and opaque.

---

# 63. Full Worked Example: A Complete Run, End to End

A "new customer onboarding" workflow: webhook trigger on signup -> charge a setup fee -> parallel(send welcome email, provision account) -> join -> approval (manager sign-off for enterprise accounts) -> create shipment (welcome kit).

```text
1.  POST /webhooks/signup-trigger  --  HMAC verified (§27), delivery ID not seen before (dedup, §27)
2.  RunStarted appended (§22): runId=r1, workflowId=onboarding, definitionVersion=4 (pinned, §58)
3.  OrchestrationEngine.resume(r1) (§23) -- fresh run, replays nothing, starts at "chargeSetupFee"
4.  dispatchWithRetry(chargeSetupFee) (§49) -- ChargeCardActionExecutor.execute(), idempotencyKey=r1:chargeSetupFee:1
    -> StepSucceeded appended, output={chargeId: "ch_abc"}
5.  nextStepsAfter (§44) -> ParallelStep fans out to [sendWelcomeEmail, provisionAccount] (§45)
6.  Both dispatched to their own worker pools (§39) -- sendWelcomeEmail succeeds; provisionAccount succeeds
7.  JoinStep.isReady() (§45) -- expectedIncomingCount=2, both succeeded -> ready, dispatched
8.  ApprovalStep dispatched (§53) -- transitions to WAITING, notifyApprovers() sent, thread released immediately
    -- ... two days pass; NOTHING is polling, blocked, or holding a connection for those two days ...
9.  POST /runs/r1/steps/approval/resume {"approved": true}  (§14, §53)
    -> StepSucceeded appended -> resume(r1) called -- the SAME method step 3 called
10. nextStepsAfter -> createShipment dispatched -> StepSucceeded -> RunCompleted appended (§22)
```

Every numbered line traces to a section this guide built real code for — including the two-day gap in step 8, which is exactly the point: nothing in this trace requires a thread, a lock, or a connection to survive that gap, only a durable log entry and a webhook that arrives whenever it arrives.

---

# 64. Final Architecture Diagram

```text
                         Definition Service (§13-§18)         Trigger Service (§25-§27)
                       WorkflowDefinition, versioned (§58)    Webhook / Cron / Event / Manual
                                    |                                    |
                                    +-------------------+-----------------+
                                                         v
                                          Orchestration Engine (§20, §23, §44-§46)
                                    reads/appends the per-run Event Log (§22), partitioned by
                                    runId across instances (§60), pins each run to its definition version
                                                         |
                          +------------------------------+------------------------------+
                          v                              v                              v
                Worker Pool ("http")           Worker Pool ("email")          Worker Pool ("custom")
                rate-limited per tenant (§61)   autoscaled by queue depth (§62)   sandboxed (§67)
                          |                              |                              |
                          +------------------------------+------------------------------+
                                                         v
                                         ActionExecutorRegistry (§37) --> External systems
                                                         |
                                    RetryPolicy (§48-§49) + IdempotencyKey (§50) on every call
                                                         |
                                On permanent failure: Dead-Letter Queue (§51) --> Saga Compensation (§55-§56)
```

---

# 65. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **State** | `StepStatus`/`RunStatus` (§33-§34) | Legal transitions live in one place, enforced polymorphically, never a scattered `if/else` on a raw enum. |
| **Strategy** | `ActionExecutor` (§37) | Each connector type is an interchangeable implementation, selected at runtime by `connectorType()`. |
| **Command** | `WorkflowEvent` (§22) | Every event is a small, self-contained, replayable record of "what happened" — exactly what makes replay (§23) possible. |
| **Event Sourcing** | The event log as the source of truth (§22-§23) | Current state is always *derived*, never independently stored, which is what unifies crash recovery and human-approval resume into one mechanism (§53). |
| **Saga** | `CompensableStep` (§55) | Atomicity across real-world side effects, achieved with domain-specific inverse operations instead of a database rollback. |
| **Composite (conceptually)** | `SubWorkflowStep`, `ForEachStep.body()` (§17, §46) | A workflow step can itself contain another step (or another whole workflow), executed by the same generic machinery as any leaf step. |
| **Facade** | `OrchestrationEngine` (§23) | Triggers, resumes, and retries all funnel through one entry point that hides the event log, the registry, and the definition registry from every caller. |
| **Observer (conceptually)** | The internal event-bus trigger (§25) | A workflow run starting in reaction to "something else happened" is the same publish/listen shape [the JIRA-style guide's Observer-based notifications](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) already builds. |

---

# 66. SOLID Principles Applied

- **Single Responsibility**: `WorkflowValidator` only validates; `EventLog` only appends and reads; `ExpressionEvaluator` only resolves paths against a context. None of them know how to dispatch a step or retry a failure.
- **Open/Closed**: adding a new connector (§37) or a new `RetryPolicy` implementation requires zero changes to `OrchestrationEngine` — both are consumed purely through their interfaces.
- **Liskov Substitution**: any `ActionExecutor` is fully substitutable wherever the interface type is used (§37's registry) — `HttpActionExecutor` and `EmailActionExecutor` (§38) are interchangeable from the orchestrator's point of view, differing only in behavior, never in contract.
- **Interface Segregation**: `ActionExecutor` exposes exactly two methods; `CompensableStep` (§55) is a *separate*, optional interface an executor implements only if it actually has something to compensate — an executor with nothing to undo (a pure read) is never forced to implement a meaningless `compensate()`.
- **Dependency Inversion**: `OrchestrationEngine` (§23) depends on `EventLog`, `ActionExecutorRegistry`, and `WorkflowDefinitionRegistry` — all interfaces/registries injected through its constructor — never on `FileEventLog` or any other concrete storage implementation directly.

---

# 67. Common Mistakes When Building This Yourself

- **Treating in-memory execution state as authoritative** (§21) — the single most common design flaw; if a crash can lose progress that was never durably logged, every other guarantee this guide makes collapses.
- **Retrying without an idempotency key** (§50) — a retry that reaches a downstream system successfully both times produces a duplicate side effect, silently, in exactly the cases that matter most (payments, shipments).
- **Blocking a thread for a human-approval step** (§52-§53) — holding a worker-pool thread, a database connection, or a lock for however long a human takes to respond exhausts a bounded resource for no benefit; the `WAITING` state exists specifically so nothing is held.
- **A shared worker pool across every connector type or every tenant** (§39, §61) — one slow integration, or one noisy tenant, silently starves every unrelated one sharing the same pool.
- **Compensating in forward order instead of reverse** (§55) — undoing effects in the order they happened, rather than the reverse, can attempt to compensate a step whose *own* prerequisite (an earlier step) hasn't been undone yet, when the compensation logic assumed it would be.
- **Letting an in-flight run silently pick up a newly-published definition version** (§57-§58) — a run's graph shifting underneath it mid-execution is a correctness bug users experience as "the workflow ran differently for no reason," often days after the version change that actually caused it.
- **A workflow definition that permits a cycle** (§18) — without validating this at publish time, a mistake in a condition's edges can produce a run that loops forever, discovered only by an on-call engineer staring at a suspiciously long-running run.

---

# 68. Testing Strategy

- **`WorkflowValidator`** (§18): a definition with a genuine cycle is rejected; a definition with an unreachable step is rejected; a definition with a dangling edge (pointing at a non-existent step) is rejected; a valid, non-trivial DAG (including one with a `JoinStep`) passes cleanly.
- **`OrchestrationEngine.resume`** (§23): replaying a log containing only `RunStarted` + one `StepSucceeded` correctly resumes from the step *after* the succeeded one, never re-dispatching the succeeded one; replaying a log ending in `RunCompleted` is a no-op.
- **Retry and idempotency** (§48-§50): a step that fails twice then succeeds ends with exactly one `StepSucceeded` event and the correct `attempt` count; the idempotency key passed to the connector is identical across all attempts of the same logical step execution.
- **The Saga path** (§55-§56): a three-step run where step 3 permanently fails results in steps 1 and 2's `compensate()` being called, in strict reverse order, with the original step's own output as input to its compensation.
- **Approval suspend/resume** (§52-§53): a run reaches `WAITING` and genuinely returns control (no thread, connection, or lock held); calling the resume endpoint after the process has *restarted* (simulating exactly the crash-recovery machinery, §23) still correctly continues the run.
- **Versioning** (§57-§58): a run started under version 3 continues to resolve version 3 even after versions 4 and 5 are published mid-run; a brand-new trigger fire always resolves the latest version.

---

# 69. Suggested Future Enhancements

- **A visual workflow builder** compiling directly to the §17 DSL, with `WorkflowValidator` (§18) run live as the user edits, surfacing a cycle or an unreachable step immediately rather than at publish time.
- **A richer expression language** — comparison/boolean operators, simple string functions — while still deliberately stopping short of a general-purpose scripting language, for the same security-surface reason §41 gives.
- **A sandboxed custom-code step** (arbitrary user-supplied logic) — genuinely useful, and genuinely dangerous without strict resource limits, network egress controls, and a dedicated, isolated worker pool (already named as a placeholder in §39's diagram).
- **Fuzzy, non-quiescent checkpointing of the event log** for very long-running workflows with thousands of events, mirroring the exact tradeoff the [TinyDB Storage Engine guide's checkpoint section](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) names for its own recovery-time bound.
- **A workflow-of-workflows dependency graph**, so publishing a breaking change to a sub-workflow (§17's `SubWorkflowStep`) can warn about every parent workflow that references it, before publishing, not after something downstream breaks.
- **Per-tenant SLAs on run completion time**, feeding directly into §62's autoscaling signal so a tenant paying for a faster tier gets prioritized queue placement, not just a bigger pool.

---

# 70. Progressive Interview Question Set

1. Why must a workflow definition be acyclic, and how do you detect a cycle in a graph that also has shared sub-paths (two branches feeding into one join)?
2. Walk through exactly what happens, event by event, when a worker process crashes between two steps.
3. Why does redoing a step during crash recovery need to be idempotent, and how does an idempotency key make a *downstream connector* idempotent too, not just your own engine?
4. Explain why a human-approval step doesn't block a thread, and name the one method that both crash recovery and approval-resume both call.
5. Why does a `JoinStep` need an explicitly declared `expectedIncomingCount` rather than inferring it from the graph?
6. What's the difference between retrying a step and compensating a step, and why does a Saga need to walk completed steps in *reverse* order?
7. Why must an in-flight run stay pinned to the workflow definition version it started with, even after a newer version is published?
8. What specifically breaks first as this system scales to millions of runs a day, and what's the standard fix for each bottleneck you name?
9. Why does adding a new connector type never require modifying `OrchestrationEngine`, concretely — which interface makes that true?
10. If asked to add a "wait until a specific timestamp" step (not tied to any external event), how would you implement it using pieces this guide already built, without introducing a new blocking mechanism?

---

# 71. Final Takeaway

A workflow automation system is not, at its core, a scheduling problem or an integrations problem — it is a **durability** problem wearing a graph-execution costume. Every genuinely hard piece of this design — surviving a crash mid-run, resuming a three-day approval without holding a thread, retrying a flaky call without duplicating its side effect, rolling back a partially-completed run — reduces to the same single decision made once, early, in §22: **treat an append-only, durable event log as the only source of truth, and derive everything else from it.** The DAG, the State pattern, the Strategy-based connectors, and the Saga compensation are all real, necessary pieces of engineering — but none of them would be *safe* pieces of engineering without that one durability decision underneath all of them.

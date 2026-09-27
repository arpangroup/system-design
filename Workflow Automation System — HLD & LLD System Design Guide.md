# Workflow Automation System — HLD & LLD System Design Guide

## 1. System Design Interview Question

### Primary Question

> **Design a Workflow Automation System**
>
> Build a distributed workflow automation platform where users can create workflows consisting of multiple steps/actions, define conditions and schedules, and execute those workflows automatically.
>
> The system should support:
>
> - Workflow creation and versioning
> - Multiple workflow steps
> - Sequential and parallel execution
> - Conditional branching
> - Scheduled workflows
> - Event-triggered workflows
> - HTTP/API actions
> - Email/SMS/notification actions
> - Delays and timers
> - Retries
> - Failure handling
> - Workflow cancellation
> - Execution history
> - Monitoring
> - Idempotent execution
> - Horizontal scalability
>
> Design the system using **SOLID principles**, clean architecture, extensible interfaces, and appropriate design patterns.

---

# 2. Interview Follow-Ups

The interviewer can progressively ask:

### Follow-up 1 — Requirements

- What are the functional requirements?
- What are the non-functional requirements?
- Who are the users?
- What exactly is a workflow?

### Follow-up 2 — Workflow Model

- How would you represent a workflow?
- How do you represent workflow steps?
- How do you represent dependencies?
- Can steps execute in parallel?

### Follow-up 3 — Execution

- How does a workflow actually execute?
- How do workers pick tasks?
- How do you handle failures?

### Follow-up 4 — Scheduling

- How would you support:
  - `run every day`
  - `run every Monday`
  - `run at 10 AM`
  - `run after 30 minutes`

### Follow-up 5 — Distributed System

- How do you scale workers?
- How do you prevent duplicate execution?
- What happens if a worker crashes?

### Follow-up 6 — Reliability

- How do you implement retries?
- What is exponential backoff?
- What happens after maximum retries?

### Follow-up 7 — Consistency

- How do you guarantee exactly-once execution?
- Is exactly-once actually possible?
- How do you implement idempotency?

### Follow-up 8 — Workflow Evolution

- What happens when a workflow is modified while an execution is already running?
- How do you version workflows?

### Follow-up 9 — Advanced

- How would you implement:
  - parallel branches?
  - loops?
  - conditions?
  - human approval?
  - compensation?
  - long-running workflows?

### Follow-up 10 — Multi-Tenancy

- How can thousands of organizations use the same platform?
- How do you isolate tenants?

---

# 3. First Clarify the Scope

A workflow can be represented as:

```text
Trigger
   |
   v
Step A
   |
   v
Condition
  / \
Yes  No
 |    |
 v    v
Step B Step C
  \    /
   \  /
    v
 Step D
```

For example:

```text
New Order Created
        |
        v
Validate Order
        |
        v
Payment Successful?
      /       \
    YES        NO
     |          |
     v          v
Send Email   Send Failure
     |
     v
Create Shipment
     |
     v
Wait 24 Hours
     |
     v
Send Feedback Email
```

---

# 4. Functional Requirements

## 4.1 Workflow Management

The system should allow users to:

- Create workflows
- Update workflows
- Delete workflows
- Enable/disable workflows
- Clone workflows
- Publish workflows
- Version workflows

---

## 4.2 Workflow Definition

A workflow contains:

```text
Workflow
    |
    +-- Trigger
    |
    +-- Nodes
    |
    +-- Edges
    |
    +-- Variables
    |
    +-- Error Policy
```

Example:

```json
{
  "workflowId": "wf-1001",
  "name": "Order Processing",
  "version": 3,
  "trigger": {
    "type": "EVENT",
    "event": "ORDER_CREATED"
  },
  "nodes": [
    {
      "id": "validate",
      "type": "HTTP"
    },
    {
      "id": "payment",
      "type": "CONDITION"
    },
    {
      "id": "email",
      "type": "EMAIL"
    }
  ]
}
```

---

# 5. Supported Trigger Types

Initially support:

```text
MANUAL
SCHEDULE
EVENT
WEBHOOK
API
```

Examples:

### Schedule

```text
Every day at 10 AM
```

### Event

```text
ORDER_CREATED
```

### Webhook

```text
POST /webhooks/customer
```

### Manual

```text
User clicks "Run"
```

---

# 6. Supported Action Types

The architecture should allow new actions without modifying the workflow engine.

Examples:

```text
HTTP_REQUEST
EMAIL
SMS
PUSH_NOTIFICATION
DATABASE
WEBHOOK
WAIT
CONDITION
TRANSFORM
LOG
APPROVAL
```

Future:

```text
SLACK
TEAMS
JIRA
SALESFORCE
KAFKA
AWS
GCP
AI_AGENT
```

This is where the **Open/Closed Principle** becomes important.

---

# 7. Non-Functional Requirements

## Scalability

Target example:

```text
10M workflows
1M active workflows
100K executions/minute
10K concurrent executions
```

These numbers are interview assumptions and should be adjusted according to the expected workload.

---

## Availability

Target:

```text
99.9%+
```

for the control plane.

Execution availability may have different guarantees depending on the action.

---

## Reliability

The system should survive:

- Worker crash
- Scheduler crash
- Message broker failure
- Database failover
- Network timeout
- External API failure

---

## Durability

Workflow definitions and execution state must survive service restarts.

---

## Observability

Track:

```text
Workflow started
Step started
Step completed
Step failed
Retry
Workflow completed
Workflow failed
Workflow cancelled
```

---

# 8. High-Level Architecture

```text
                        +--------------------+
                        |      Client        |
                        | Web / Mobile / API |
                        +---------+----------+
                                  |
                                  v
                        +--------------------+
                        |     API Gateway    |
                        +---------+----------+
                                  |
                +-----------------+-----------------+
                |                                   |
                v                                   v
       +-------------------+               +-------------------+
       | Workflow Service  |               | Trigger Service   |
       +---------+---------+               +---------+---------+
                 |                                   |
                 v                                   v
       +-------------------+               +-------------------+
       | Workflow DB       |               | Scheduler         |
       +-------------------+               +---------+---------+
                                                     |
                                                     v
                                           +-------------------+
                                           | Message Broker    |
                                           | Kafka/SQS/etc.    |
                                           +---------+---------+
                                                     |
                         +---------------------------+--------------------+
                         |                           |                    |
                         v                           v                    v
                 +---------------+           +---------------+    +---------------+
                 | Worker        |           | Worker        |    | Worker        |
                 | Instance 1   |           | Instance 2   |    | Instance N   |
                 +-------+-------+           +-------+-------+    +-------+-------+
                         |                           |                    |
                         +---------------------------+--------------------+
                                                     |
                                                     v
                                          +----------------------+
                                          | Execution State DB   |
                                          +----------------------+

                                                     |
                                                     v
                                          +----------------------+
                                          | Observability        |
                                          | Metrics/Logs/Traces  |
                                          +----------------------+
```

---

# 9. Separate Control Plane and Data Plane

A useful architectural decision is to divide the system into two logical planes.

## Control Plane

Responsible for:

```text
Workflow creation
Workflow update
Workflow validation
Workflow publishing
Workflow versioning
Configuration
Authentication
Authorization
```

Services:

```text
Workflow API
Workflow Repository
Workflow Validator
Version Manager
```

---

## Data Plane

Responsible for:

```text
Workflow triggering
Workflow execution
Task execution
Retries
Scheduling
State transitions
```

Services:

```text
Trigger Service
Scheduler
Execution Engine
Task Queue
Workers
```

This separation makes scaling easier.

---

# 10. Core Data Model

## Workflow

```text
Workflow
---------
id
tenantId
name
description
status
currentVersion
createdBy
createdAt
updatedAt
```

---

## WorkflowVersion

```text
WorkflowVersion
---------------
id
workflowId
version
definition
status
createdAt
publishedAt
```

Never modify a published workflow version.

Instead:

```text
Workflow
   |
   +-- Version 1
   |
   +-- Version 2
   |
   +-- Version 3
```

---

# 11. Workflow Node

```text
WorkflowNode
------------
id
workflowVersionId
nodeKey
type
configuration
```

Example:

```json
{
  "nodeKey": "sendEmail",
  "type": "EMAIL",
  "configuration": {
    "to": "{{customer.email}}",
    "template": "ORDER_CONFIRMED"
  }
}
```

---

# 12. Workflow Edge

An edge represents a transition.

```text
WorkflowEdge
------------
id
workflowVersionId
sourceNode
targetNode
condition
```

Example:

```text
Validate
   |
   v
Payment
 /     \
YES     NO
 |       |
Email   Failure
```

Edges:

```text
Validate -> Payment

Payment -> Email
condition = payment.success == true

Payment -> Failure
condition = payment.success == false
```

---

# 13. Workflow Execution

Every workflow invocation gets an execution ID.

```text
WorkflowExecution
-----------------
id
workflowId
workflowVersion
status
triggerType
startedAt
completedAt
context
```

Example:

```text
executionId = exec-123
workflowId   = wf-100
version      = 4
status       = RUNNING
```

---

# 14. Step Execution

Each node execution should also have its own record.

```text
StepExecution
-------------
id
executionId
nodeId
status
attempt
startedAt
completedAt
input
output
error
```

Example:

```text
Execution
    |
    +-- Step A -> SUCCESS
    |
    +-- Step B -> SUCCESS
    |
    +-- Step C -> FAILED
              |
              +-- Retry 1
              +-- Retry 2
              +-- Retry 3
```

---

# 15. Database Design

A relational database works well for metadata and durable state.

Possible PostgreSQL schema:

```text
workflow
workflow_version
workflow_node
workflow_edge

workflow_execution
step_execution

workflow_trigger
workflow_schedule

workflow_variable
workflow_event

retry_policy
execution_event
```

---

# 16. Why Store Workflow Definitions as Graphs?

A workflow is naturally a directed graph.

```text
       A
       |
       v
       B
      / \
     v   v
     C   D
      \ /
       v
       E
```

Therefore:

```text
V = nodes
E = edges
```

The workflow engine can perform graph traversal.

---

# 17. Workflow Execution Engine

The execution engine is the heart of the system.

Conceptually:

```text
WorkflowExecutionEngine
        |
        v
Load Workflow Definition
        |
        v
Find Ready Nodes
        |
        v
Create Task
        |
        v
Submit Task
        |
        v
Worker Executes
        |
        v
Persist Result
        |
        v
Evaluate Next Nodes
        |
        v
Repeat
```

---

# 18. Ready Node Calculation

Suppose:

```text
A -> B -> C
```

Initially:

```text
Ready = A
```

After A succeeds:

```text
Ready = B
```

After B succeeds:

```text
Ready = C
```

For parallel execution:

```text
       A
      / \
     B   C
      \ /
       D
```

After A:

```text
Ready = B, C
```

B and C can execute concurrently.

D becomes ready only after its dependencies are satisfied.

---

# 19. Task Queue

Do not execute everything synchronously inside the API request.

Instead:

```text
API
 |
 v
Create Execution
 |
 v
Queue Task
 |
 v
Worker
 |
 v
Execute
```

Possible technologies:

```text
Kafka
RabbitMQ
SQS
Redis Streams
```

---

# 20. Task Message

Example:

```json
{
  "taskId": "task-100",
  "executionId": "exec-123",
  "workflowId": "wf-1",
  "workflowVersion": 5,
  "nodeId": "sendEmail",
  "attempt": 1
}
```

Important:

The message should contain enough information to identify the task, but the worker should retrieve authoritative workflow/execution state from durable storage where appropriate.

---

# 21. Worker Architecture

A worker should not contain workflow-specific logic.

Instead:

```text
Worker
   |
   v
Task Executor
   |
   v
Action Registry
   |
   +-- EmailAction
   +-- HttpAction
   +-- SmsAction
   +-- WaitAction
   +-- ConditionAction
```

---

# 22. SOLID Principle — Open/Closed

Bad design:

```java
if (type == EMAIL) {
   ...
} else if (type == SMS) {
   ...
} else if (type == HTTP) {
   ...
}
```

Every new action modifies the worker.

Better:

```java
public interface ActionExecutor {

    ActionType supports();

    ActionResult execute(ActionContext context);
}
```

Implementations:

```java
class EmailActionExecutor implements ActionExecutor {
}

class SmsActionExecutor implements ActionExecutor {
}

class HttpActionExecutor implements ActionExecutor {
}
```

Adding a new action:

```java
class SlackActionExecutor implements ActionExecutor {
}
```

No modification to the core engine is required.

---

# 23. Action Registry

```java
public interface ActionRegistry {

    ActionExecutor get(ActionType type);
}
```

Implementation:

```java
class DefaultActionRegistry implements ActionRegistry {

    private final Map<ActionType, ActionExecutor> executors;

    @Override
    public ActionExecutor get(ActionType type) {
        return executors.get(type);
    }
}
```

---

# 24. Liskov Substitution Principle

Every `ActionExecutor` should obey the same contract.

```java
interface ActionExecutor {

    ActionResult execute(ActionContext context);
}
```

The engine should not care whether the executor is:

```text
Email
HTTP
SMS
Database
Webhook
```

All must be replaceable through the abstraction.

---

# 25. Interface Segregation

Avoid:

```java
interface WorkflowService {

    create();

    update();

    delete();

    execute();

    schedule();

    retry();

    cancel();

    approve();

    sendEmail();
}
```

Instead:

```java
interface WorkflowRepository {
}

interface WorkflowExecutor {
}

interface WorkflowScheduler {
}

interface WorkflowCanceller {
}

interface RetryManager {
}
```

Each interface has a focused responsibility.

---

# 26. Dependency Inversion

Bad:

```java
class WorkflowEngine {

    private KafkaProducer producer =
        new KafkaProducer(...);
}
```

Better:

```java
class WorkflowEngine {

    private final TaskPublisher taskPublisher;

    WorkflowEngine(TaskPublisher taskPublisher) {
        this.taskPublisher = taskPublisher;
    }
}
```

Infrastructure implementation:

```java
class KafkaTaskPublisher implements TaskPublisher {
}
```

The core engine depends on an abstraction.

---

# 27. Class-Level Design

Core interfaces:

```text
                +----------------------+
                | WorkflowRepository   |
                +----------+-----------+
                           |
                           v
                WorkflowRepositoryImpl


                +----------------------+
                | WorkflowEngine       |
                +----------+-----------+
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
       TaskPublisher   StateManager   ActionRegistry
             |                           |
             v                           v
        KafkaPublisher             ActionExecutor
                                       / | \
                                      /  |  \
                                     v   v   v
                                  Email HTTP SMS
```

---

# 28. Core Domain Classes

```java
class Workflow {
    private WorkflowId id;
    private String name;
    private WorkflowStatus status;
    private WorkflowVersion currentVersion;
}
```

```java
class WorkflowVersion {
    private int version;
    private List<WorkflowNode> nodes;
    private List<WorkflowEdge> edges;
}
```

```java
class WorkflowNode {
    private NodeId id;
    private NodeType type;
    private Map<String, Object> configuration;
}
```

```java
class WorkflowEdge {
    private NodeId source;
    private NodeId target;
    private Condition condition;
}
```

---

# 29. Execution Classes

```java
class WorkflowExecution {

    private ExecutionId id;
    private WorkflowId workflowId;
    private int workflowVersion;

    private ExecutionStatus status;

    private ExecutionContext context;
}
```

```java
class StepExecution {

    private StepExecutionId id;

    private ExecutionId executionId;

    private NodeId nodeId;

    private StepStatus status;

    private int attempt;
}
```

---

# 30. Execution Context

Workflow steps need to share data.

Example:

```text
Trigger
   |
   v
Fetch Customer
   |
   v
Send Email
```

Fetch Customer produces:

```json
{
  "customer": {
    "id": 100,
    "email": "user@example.com"
  }
}
```

Email uses:

```text
{{customer.email}}
```

Therefore:

```java
class ExecutionContext {

    private Map<String, Object> variables;

    public Object get(String key) {
        return variables.get(key);
    }

    public void put(String key, Object value) {
        variables.put(key, value);
    }
}
```

---

# 31. Expression Evaluation

Conditions may look like:

```text
payment.status == "SUCCESS"
```

or:

```text
order.amount > 1000
```

Create an abstraction:

```java
interface ExpressionEvaluator {

    boolean evaluate(
        String expression,
        ExecutionContext context
    );
}
```

Possible implementation:

```java
class DefaultExpressionEvaluator
        implements ExpressionEvaluator {
}
```

The workflow engine does not depend on a particular expression language.

---

# 32. Condition Executor

```java
class ConditionActionExecutor
        implements ActionExecutor {

    private final ExpressionEvaluator evaluator;

    @Override
    public ActionResult execute(ActionContext context) {

        boolean result =
            evaluator.evaluate(
                context.expression(),
                context.executionContext()
            );

        return ActionResult.success(result);
    }
}
```

---

# 33. Retry Architecture

Suppose:

```text
HTTP API
```

returns:

```text
500
```

The worker should not immediately fail the workflow.

Retry policy:

```text
Attempt 1
   |
   wait 1 sec
   |
Attempt 2
   |
   wait 2 sec
   |
Attempt 3
   |
   wait 4 sec
   |
Attempt 4
   |
   FAILED
```

---

# 34. Retry Policy

```java
class RetryPolicy {

    private int maxAttempts;

    private Duration initialDelay;

    private double multiplier;

    private Duration maxDelay;
}
```

Formula:

```text
delay = min(
    initialDelay * multiplier^(attempt-1),
    maxDelay
)
```

Add jitter to avoid many workers retrying simultaneously.

---

# 35. Retryable vs Non-Retryable Errors

Not every failure should be retried.

Examples:

```text
HTTP 500       -> retry
HTTP 503       -> retry
Timeout        -> retry

HTTP 400       -> usually don't retry
Invalid input  -> don't retry
Authentication -> usually don't retry
```

Define:

```java
interface RetryPolicy {

    boolean shouldRetry(
        Exception exception,
        int attempt
    );
}
```

---

# 36. Idempotency

This is one of the most important interview topics.

Suppose:

```text
Worker sends payment request
       |
       v
Payment succeeds
       |
       X
Worker crashes before recording success
```

The queue retries the task.

Now payment may happen twice.

Solution:

```text
Idempotency Key
```

Example:

```text
workflowId
+
executionId
+
nodeId
```

could produce:

```text
idempotencyKey =
wf1-exec123-payment
```

The downstream service should recognize repeated requests with the same key.

---

# 37. Exactly Once vs At Least Once

Distributed systems commonly use:

```text
At-least-once delivery
```

rather than claiming true exactly-once execution across arbitrary external systems.

Therefore:

```text
Queue
   |
   v
Worker
   |
   v
Idempotent Action
```

is a better architecture.

The workflow engine can provide durable state transitions and deduplication, while external side effects require idempotency support.

---

# 38. Worker Crash Handling

Suppose:

```text
Task status = RUNNING
```

but worker crashes.

A lease can be used:

```text
task
-----
status = RUNNING
leaseUntil = 12:05:00
workerId = worker-10
```

If:

```text
currentTime > leaseUntil
```

another worker can reclaim the task.

Use:

```text
lease + heartbeat
```

for long-running tasks.

---

# 39. Task State Machine

Define explicit states:

```text
PENDING
   |
   v
RUNNING
  / \
 v   v
SUCCESS FAILED
         |
         v
       RETRY
         |
         v
      RUNNING
```

Terminal states:

```text
SUCCESS
FAILED
CANCELLED
```

This prevents invalid transitions.

---

# 40. State Machine Design Pattern

Create:

```java
interface TaskStateHandler {

    TaskStatus status();

    TaskStatus handle(TaskContext context);
}
```

Possible handlers:

```text
PendingState
RunningState
SuccessState
FailedState
CancelledState
```

This avoids putting all transition logic inside one giant class.

---

# 41. Scheduling Architecture

For:

```text
Every day at 10 AM
```

store:

```text
schedule
---------
workflowId
cronExpression
timezone
nextExecutionTime
enabled
```

Example:

```text
0 10 * * *
```

The scheduler periodically finds:

```text
nextExecutionTime <= now
```

and publishes execution requests.

---

# 42. Distributed Scheduler

A single scheduler becomes a bottleneck.

Instead:

```text
Scheduler 1
Scheduler 2
Scheduler 3
```

All can run concurrently if work is partitioned.

Possible approaches:

```text
Database row locking
Leader election
Distributed locks
Partitioned scheduling
```

For large systems:

```text
time buckets
+
partitioned scheduler
```

can scale better than scanning the entire schedule table.

---

# 43. Event-Driven Trigger

Example:

```text
Order Service
     |
     v
Kafka
     |
     v
Trigger Service
     |
     v
Find matching workflows
     |
     v
Create executions
     |
     v
Task Queue
```

Workflow trigger:

```text
eventType = ORDER_CREATED
```

---

# 44. Webhook Trigger

Example:

```http
POST /api/v1/webhooks/wf-123
```

Flow:

```text
External System
       |
       v
API Gateway
       |
       v
Webhook Service
       |
       v
Validate Signature
       |
       v
Create Execution
       |
       v
Task Queue
```

Important:

Webhook processing should normally be asynchronous.

Return:

```http
202 Accepted
```

after durable acceptance.

---

# 45. Parallel Execution

Workflow:

```text
        A
      / | \
     B  C  D
      \ | /
        E
```

After A:

```text
B
C
D
```

can execute concurrently.

E should wait until all required dependencies complete.

The engine maintains dependency counts.

Example:

```text
E dependencies = 3

B -> completed => 2
C -> completed => 1
D -> completed => 0

E becomes READY
```

---

# 46. Failure in Parallel Branches

Consider:

```text
      A
     / \
    B   C
     \ /
      D
```

B succeeds.

C fails.

What happens?

The workflow needs an explicit policy:

```text
FAIL_FAST
WAIT_ALL
CONTINUE_SUCCESSFUL_BRANCHES
COMPENSATE
```

This should be part of workflow semantics rather than hardcoded.

---

# 47. Compensation / Saga

For long-running workflows:

```text
Reserve Inventory
       |
       v
Charge Payment
       |
       v
Create Shipment
```

If shipment fails:

```text
Cancel Payment
       |
       v
Release Inventory
```

These are compensation actions.

Model:

```text
Action
  |
  +-- execute()
  |
  +-- compensate()
```

For more flexibility:

```java
interface CompensationHandler {
    void compensate(ActionContext context);
}
```

---

# 48. Human Approval

Future node:

```text
MANUAL_APPROVAL
```

Workflow:

```text
Order
 |
 v
Payment
 |
 v
Manager Approval
 |
 +-----> APPROVED
 |
 +-----> REJECTED
```

Execution remains:

```text
WAITING_FOR_INPUT
```

until the approval arrives.

This is important for long-running workflow support.

---

# 49. Long-Running Workflow

A workflow may run for:

```text
milliseconds
seconds
hours
days
months
```

Never keep a Java thread blocked for days.

Bad:

```java
Thread.sleep(24 hours);
```

Better:

```text
Persist WAIT state
       |
       v
Create timer
       |
       v
Release worker
       |
       v
Timer fires
       |
       v
Queue next task
```

---

# 50. Timer Service

```text
Workflow Engine
      |
      v
Timer Service
      |
      v
Delayed Task
      |
      v
Task Queue
```

For very large systems, timers may be partitioned by time buckets.

---

# 51. Workflow Cancellation

User:

```text
Cancel Execution
```

The engine should:

```text
1. Mark execution CANCEL_REQUESTED
2. Stop scheduling new tasks
3. Signal running tasks where possible
4. Mark cancellable tasks
5. Persist final state
```

External operations cannot always be forcibly cancelled.

Therefore cancellation semantics must be documented.

---

# 52. Workflow Versioning

Suppose:

```text
Workflow Version 1
```

is executing.

User publishes:

```text
Version 2
```

Should the existing execution switch to V2?

Normally:

```text
Execution 100
    -> Workflow Version 1

New Execution 101
    -> Workflow Version 2
```

This makes executions deterministic.

---

# 53. API Design

## Create Workflow

```http
POST /api/v1/workflows
```

## Get Workflow

```http
GET /api/v1/workflows/{id}
```

## Update Workflow

```http
PUT /api/v1/workflows/{id}
```

## Publish

```http
POST /api/v1/workflows/{id}/publish
```

## Execute

```http
POST /api/v1/workflows/{id}/execute
```

## Cancel

```http
POST /api/v1/executions/{id}/cancel
```

## Execution Status

```http
GET /api/v1/executions/{id}
```

---

# 54. API Idempotency

For:

```http
POST /workflows/{id}/execute
```

the client may send:

```http
Idempotency-Key: abc-123
```

Store:

```text
tenantId
idempotencyKey
executionId
expiresAt
```

Repeated requests return the same execution.

---

# 55. Multi-Tenancy

Every major entity should contain:

```text
tenantId
```

Example:

```text
workflow
---------
id
tenantId
name
```

All queries must enforce tenant isolation.

Never allow:

```text
tenant A -> workflow of tenant B
```

---

# 56. Authorization

Possible roles:

```text
ADMIN
WORKFLOW_EDITOR
WORKFLOW_OPERATOR
VIEWER
```

Permissions:

```text
workflow:create
workflow:update
workflow:publish
workflow:execute
workflow:cancel
workflow:view
```

---

# 57. Security

Protect:

```text
Workflow APIs
Webhook endpoints
Secrets
Credentials
Execution data
Logs
```

Secrets should never be stored as plaintext inside workflow definitions.

Instead:

```text
Workflow
   |
   v
secretReference
   |
   v
Secret Manager
```

Examples:

```text
AWS Secrets Manager
HashiCorp Vault
Kubernetes Secrets
Cloud Secret Manager
```

---

# 58. Expression Security

Never execute arbitrary code supplied by users.

Avoid:

```text
eval(userInput)
```

Use a restricted expression language:

```text
amount > 1000
status == "SUCCESS"
customer.country == "IN"
```

The evaluator should have:

```text
No filesystem access
No network access
No arbitrary reflection
No code execution
```

---

# 59. Observability

Metrics:

```text
workflow.executions.total
workflow.executions.success
workflow.executions.failed

workflow.step.duration
workflow.step.failure

queue.depth
queue.processing.time

worker.active
worker.failed
```

Distributed tracing:

```text
Trace
 |
 +-- API
 |
 +-- Workflow Engine
 |
 +-- Queue
 |
 +-- Worker
 |
 +-- External API
```

Use a correlation ID:

```text
X-Correlation-ID
```

---

# 60. Execution History

A UI should show:

```text
Workflow: Order Processing

Execution: exec-123

10:00:01  Trigger       SUCCESS
10:00:02  Validate      SUCCESS
10:00:03  Payment       SUCCESS
10:00:03  Email         RUNNING
10:00:04  Email         SUCCESS
10:00:04  Shipment      SUCCESS

Final Status: SUCCESS
```

---

# 61. Dead Letter Queue

After maximum retries:

```text
Task
 |
 +-- retry
 +-- retry
 +-- retry
 |
 v
DLQ
```

DLQ should allow:

```text
Inspect
Retry
Replay
Discard
```

But replay must respect idempotency.

---

# 62. Workflow Replay

For debugging:

```text
Execution 100 failed
```

User selects:

```text
Replay from Step C
```

The system creates:

```text
Execution 101
```

with:

```text
parentExecutionId = 100
startNode = C
```

Avoid mutating the historical execution.

---

# 63. Complete Class Diagram

```text
+----------------------+
|      Workflow        |
+----------------------+
| id                   |
| tenantId             |
| name                 |
| status               |
+----------+-----------+
           |
           | 1..*
           v
+----------------------+
|   WorkflowVersion    |
+----------------------+
| version              |
| definition           |
+----------+-----------+
           |
           +--------------------+
           |                    |
           v                    v
+------------------+   +------------------+
| WorkflowNode     |   | WorkflowEdge     |
+------------------+   +------------------+
| id               |   | sourceNode       |
| type             |   | targetNode       |
| configuration    |   | condition        |
+------------------+   +------------------+


+--------------------------+
| WorkflowExecution        |
+--------------------------+
| id                       |
| workflowId               |
| workflowVersion          |
| status                   |
| context                  |
+-------------+------------+
              |
              | 1..*
              v
+--------------------------+
| StepExecution            |
+--------------------------+
| id                       |
| nodeId                   |
| status                   |
| attempt                  |
| input                    |
| output                   |
+--------------------------+


+--------------------------+
| WorkflowEngine           |
+--------------------------+
| execute()                |
| scheduleNext()           |
| handleResult()           |
+-------------+------------+
              |
      +-------+-------+----------------+
      |               |                |
      v               v                v
+-----------+   +-----------+   +--------------+
| Scheduler |   | TaskQueue |   | StateManager |
+-----------+   +-----------+   +--------------+
                                  |
                                  v
                           +--------------+
                           | Repository   |
                           +--------------+


+---------------------------+
| ActionRegistry            |
+-------------+-------------+
              |
       +------+------+------+
       |      |      |      |
       v      v      v      v
     HTTP   Email    SMS   Wait
```

---

# 64. Important Design Patterns

## Strategy Pattern

Used for:

```text
RetryPolicy
ActionExecutor
SchedulingStrategy
FailurePolicy
```

---

## Factory Pattern

Used for:

```text
ActionExecutor creation
```

```java
ActionExecutor executor =
    actionFactory.create(node.getType());
```

---

## Command Pattern

Each workflow action can be represented as a command:

```text
ExecuteEmailCommand
ExecuteHttpCommand
ExecuteSmsCommand
```

---

## State Pattern

Used for:

```text
WorkflowExecution state
Task state
```

---

## Observer / Event Pattern

Used for:

```text
Execution events
Notifications
Audit events
Metrics
```

---

## Chain of Responsibility

Useful for:

```text
Authentication
Authorization
Validation
Rate limiting
Idempotency
```

---

# 65. Complete Execution Flow

Suppose:

```text
Workflow:
Order Created
    |
    v
Validate Order
    |
    v
Payment
    |
    v
Send Email
```

### Step 1

Client creates workflow.

```text
POST /workflows
```

---

### Step 2

Workflow is validated.

```text
Graph validation
Configuration validation
Expression validation
```

---

### Step 3

Workflow is published.

```text
Version = 1
```

---

### Step 4

Order event arrives.

```text
ORDER_CREATED
```

---

### Step 5

Trigger service finds matching workflow.

---

### Step 6

Create:

```text
WorkflowExecution
```

---

### Step 7

Find initial node.

```text
Validate Order
```

---

### Step 8

Publish task.

```text
TaskQueue
```

---

### Step 9

Worker consumes task.

---

### Step 10

Worker gets:

```text
ActionExecutor
```

from:

```text
ActionRegistry
```

---

### Step 11

Execute action.

---

### Step 12

Persist result.

```text
StepExecution = SUCCESS
```

---

### Step 13

Workflow engine evaluates outgoing edges.

---

### Step 14

Next node becomes READY.

---

### Step 15

Repeat until:

```text
SUCCESS
```

or:

```text
FAILED
```

or:

```text
CANCELLED
```

---

# 66. Transaction Boundaries

A critical design question:

> Should database update and message publishing happen in the same transaction?

Naive implementation:

```text
DB update
   |
   v
Publish Kafka
```

Failure between them can create inconsistency.

Example:

```text
DB says task READY
Kafka publish fails
```

The task may never execute.

---

# 67. Transactional Outbox Pattern

Use:

```text
Database Transaction
        |
        +-- Update Task
        |
        +-- Insert Outbox Event
```

Then:

```text
Outbox Publisher
       |
       v
Kafka
```

Architecture:

```text
+----------------------+
| Database             |
|                      |
| Task State            |
| Outbox                |
+----------+-----------+
           |
           v
   Outbox Publisher
           |
           v
        Kafka
```

This is an important distributed-system interview topic.

---

# 68. Outbox Table

```text
outbox_event
------------
id
event_type
aggregate_id
payload
status
created_at
published_at
```

Publisher:

```text
SELECT unpublished events
       |
       v
Publish Kafka
       |
       v
Mark published
```

The publisher itself must also tolerate duplicates.

---

# 69. Optimistic Locking

Workflow execution can be updated by multiple workers.

Use:

```text
version
```

Example:

```text
execution.version = 10
```

Update:

```sql
UPDATE workflow_execution
SET status = 'RUNNING',
    version = 11
WHERE id = ?
AND version = 10;
```

If:

```text
rows_updated = 0
```

another worker already modified it.

---

# 70. Distributed Locking

Locks may be useful for:

```text
Scheduler leadership
Task claiming
Rare coordination operations
```

But avoid using distributed locks everywhere.

Prefer:

```text
Database constraints
Optimistic locking
Atomic state transitions
Partition ownership
Idempotency
```

where possible.

---

# 71. Scaling Strategy

## API Layer

Stateless:

```text
API 1
API 2
API 3
...
```

Behind:

```text
Load Balancer
```

---

## Workflow Engine

Stateless where possible.

Persist state externally.

```text
Engine 1
Engine 2
Engine 3
```

---

## Workers

Horizontally scalable:

```text
Worker 1
Worker 2
...
Worker N
```

Scale based on:

```text
Queue depth
Task latency
CPU
Memory
```

---

# 72. Queue Partitioning

Large scale:

```text
Partition 0
Partition 1
Partition 2
...
Partition N
```

Partition key could be:

```text
tenantId
workflowId
executionId
```

Choice depends on ordering requirements.

If ordering is required per execution:

```text
partitionKey = executionId
```

---

# 73. Backpressure

Suppose:

```text
Incoming tasks = 100K/sec
Worker capacity = 20K/sec
```

Queue grows.

The system should not overload downstream services.

Use:

```text
Rate limiting
Concurrency limits
Queue backpressure
Circuit breakers
Bulkheads
```

---

# 74. Per-Workflow Concurrency

Example:

```text
Workflow:
Send Marketing Emails
```

You may configure:

```text
maxConcurrentExecutions = 100
```

The scheduler must not create unlimited concurrent executions.

---

# 75. External API Protection

Suppose:

```text
Payment API
```

has limit:

```text
100 requests/sec
```

Workflow workers need:

```text
RateLimiter
```

Architecture:

```text
Worker
  |
  v
RateLimiter
  |
  v
Payment API
```

---

# 76. Circuit Breaker

If an external service continuously fails:

```text
Worker
  |
  v
Circuit Breaker
  |
  v
External API
```

States:

```text
CLOSED
   |
   v
OPEN
   |
   v
HALF_OPEN
   |
   v
CLOSED
```

This prevents retry storms.

---

# 77. Caching

Potential caches:

```text
Workflow definition cache
Tenant configuration cache
Action configuration cache
Secret metadata cache
```

Do not blindly cache mutable execution state.

Workflow versions are especially cache-friendly because published versions are immutable.

---

# 78. Workflow Validation

Before publishing:

```text
Validate workflow
      |
      +-- Graph valid?
      +-- No cycles?
      +-- Trigger valid?
      +-- Node configuration valid?
      +-- Expressions valid?
      +-- Required secrets exist?
      +-- Reachable terminal node?
```

Depending on supported semantics, cycles may be allowed only for explicit loop nodes.

---

# 79. Graph Validation

Detect:

```text
Disconnected nodes
Invalid edges
Missing start node
Invalid terminal nodes
Unexpected cycles
```

Algorithm:

```text
DFS / BFS
Topological Sort
Cycle Detection
```

For a DAG workflow:

```text
Topological Sort
```

can validate ordering.

---

# 80. Workflow DSL

Instead of storing only arbitrary JSON, define a workflow model.

Example:

```yaml
workflow:
  name: order-processing

trigger:
  type: EVENT
  event: ORDER_CREATED

steps:

  - id: validate
    type: HTTP

  - id: payment
    type: HTTP

  - id: successEmail
    type: EMAIL

edges:

  - from: validate
    to: payment

  - from: payment
    to: successEmail
    condition: payment.status == "SUCCESS"
```

This can later become a visual workflow builder.

---

# 81. Visual Workflow Builder

Frontend:

```text
+----------------------------------------+
|              Workflow                  |
|                                        |
|      [Order Created]                   |
|              |                         |
|              v                         |
|      [Validate Order]                  |
|              |                         |
|              v                         |
|         [Payment?]                     |
|          /       \                     |
|       YES         NO                   |
|        |            |                  |
|     [Email]     [Failure]              |
|                                        |
+----------------------------------------+
```

The frontend generates:

```text
nodes + edges
```

The backend validates the graph.

---

# 82. Suggested Technology Stack

For a Java implementation:

```text
Java 21
Spring Boot
PostgreSQL
Kafka
Redis
Docker
Kubernetes
Prometheus
Grafana
OpenTelemetry
```

Potential alternatives:

```text
Kafka -> RabbitMQ / SQS
Redis -> Hazelcast
PostgreSQL -> MySQL
```

The domain layer should not depend directly on these technologies.

---

# 83. Suggested Package Structure

```text
com.example.workflow
│
├── domain
│   ├── workflow
│   ├── execution
│   ├── task
│   ├── trigger
│   └── action
│
├── application
│   ├── workflow
│   ├── execution
│   ├── scheduling
│   └── retry
│
├── infrastructure
│   ├── persistence
│   ├── kafka
│   ├── redis
│   ├── scheduler
│   └── external
│
└── api
    ├── controller
    ├── request
    └── response
```

---

# 84. Clean Architecture

Dependency direction:

```text
             API
              |
              v
        Application
              |
              v
           Domain
              ^
              |
      Infrastructure
```

More precisely, infrastructure implements interfaces owned by the inner layers.

Example:

```text
Domain
  |
  +-- WorkflowRepository
  |
  +-- TaskPublisher
  |
  +-- SecretProvider

Infrastructure
  |
  +-- PostgresWorkflowRepository
  +-- KafkaTaskPublisher
  +-- VaultSecretProvider
```

---

# 85. Dependency Injection

Spring:

```java
@Service
public class WorkflowEngine {

    private final WorkflowRepository repository;
    private final TaskPublisher publisher;
    private final ActionRegistry registry;

    public WorkflowEngine(
            WorkflowRepository repository,
            TaskPublisher publisher,
            ActionRegistry registry) {

        this.repository = repository;
        this.publisher = publisher;
        this.registry = registry;
    }
}
```

The engine remains testable.

---

# 86. Unit Testing Strategy

Test:

```text
Workflow validation
Graph traversal
Condition evaluation
Retry calculation
State transitions
Idempotency
Parallel execution
Failure policies
Cancellation
```

Example:

```text
Given:
A -> B -> C

When:
A succeeds

Then:
B becomes READY
```

---

# 87. Integration Testing

Test:

```text
API
PostgreSQL
Kafka
Worker
External service mock
```

Example:

```text
POST workflow
       |
       v
Publish
       |
       v
Trigger
       |
       v
Kafka
       |
       v
Worker
       |
       v
Execution SUCCESS
```

---

# 88. Failure Testing

Simulate:

```text
Worker crash
Database unavailable
Kafka unavailable
Network timeout
External API 500
External API timeout
Duplicate message
Duplicate webhook
Scheduler crash
```

This is especially important for workflow systems.

---

# 89. Capacity Planning

Suppose:

```text
1M executions/day
```

Average:

```text
1,000,000 / 86,400
≈ 11.6 executions/sec
```

If each workflow averages:

```text
10 steps
```

then:

```text
~116 task executions/sec
```

Peak traffic might be:

```text
10x average
```

Therefore:

```text
~1,160 tasks/sec
```

would be a reasonable starting point for capacity planning under this hypothetical workload.

Always calculate based on actual requirements.

---

# 90. Hot Tenant Problem

Suppose one tenant creates:

```text
500K executions/sec
```

This can overwhelm shared infrastructure.

Solutions:

```text
Tenant quotas
Tenant-level rate limiting
Partitioning
Dedicated worker pools
Priority queues
Fair scheduling
```

---

# 91. Priority Queues

Tasks may have:

```text
HIGH
MEDIUM
LOW
```

Example:

```text
Payment confirmation -> HIGH
Analytics -> LOW
```

Workers can consume according to priority.

Be careful to avoid starvation of low-priority tasks.

---

# 92. Workflow Priority

Workflow metadata:

```text
priority
```

Execution queue:

```text
HIGH
MEDIUM
LOW
```

Scheduler can assign priority.

---

# 93. Audit Trail

Store immutable audit events:

```text
WorkflowCreated
WorkflowUpdated
WorkflowPublished
WorkflowExecuted
StepExecuted
WorkflowCancelled
WorkflowDeleted
```

Example:

```text
audit_event
-----------
id
tenantId
actor
eventType
resourceId
timestamp
metadata
```

Useful for:

```text
Compliance
Debugging
Security
Customer support
```

---

# 94. Important Consistency Decisions

Document these explicitly:

| Area | Decision |
|---|---|
| Queue delivery | At-least-once |
| Workflow version | Immutable |
| Execution state | Durable |
| External action | Idempotent |
| Retry | Configurable |
| Scheduling | Persistent |
| Cancellation | Best effort |
| Workflow update | New version |
| Task claiming | Lease/optimistic locking |
| DB + queue | Outbox |

---

# 95. Failure Matrix

| Failure | Handling |
|---|---|
| Worker crash | Lease expiry |
| Kafka unavailable | Outbox |
| DB transient failure | Retry |
| HTTP timeout | Retry |
| HTTP 400 | Fail |
| Duplicate task | Idempotency |
| Scheduler crash | Persistent schedule |
| Workflow update | Versioning |
| Long wait | Timer |
| Permanent failure | DLQ |
| External API overload | Rate limiter |
| Repeated API failure | Circuit breaker |

---

# 96. Interview Discussion: Exactly Once

Interviewer:

> Can you guarantee exactly-once execution?

Answer:

> Across the entire distributed system and arbitrary external side effects, a blanket exactly-once guarantee is generally not realistic. I would use durable state, at-least-once task delivery, idempotency keys, deduplication, transactional outbox, and explicit retry semantics to achieve effectively-once business effects where the downstream operation supports idempotency.

This is a strong system-design discussion point.

---

# 97. Interview Discussion: Why Kafka?

Possible answer:

Kafka provides:

```text
High throughput
Partitioning
Durability
Consumer groups
Replay
Ordering within partition
```

But it isn't mandatory.

For lower-scale task queues:

```text
RabbitMQ
SQS
Redis Streams
```

may be simpler.

The architecture should abstract the messaging layer.

---

# 98. Interview Discussion: Why PostgreSQL?

PostgreSQL provides:

```text
ACID
Transactions
Strong consistency
JSONB
Indexes
Constraints
Row-level locking
```

Workflow metadata and execution state are relational and benefit from transactions.

At extreme scale, parts of execution/event storage can be moved to specialized stores.

---

# 99. Future Enhancement — Workflow-as-Code

Allow developers to define:

```java
workflow()
    .trigger(orderCreated())
    .step(validate())
    .step(payment())
    .condition(...)
    .step(sendEmail());
```

This enables:

```text
Git versioning
Code review
CI/CD
Testing
```

while the visual builder remains available.

---

# 100. Future Enhancement — AI-Assisted Workflows

Possible capabilities:

```text
Natural language -> workflow
```

Example:

> "Whenever a customer places an order above ₹10,000, notify the manager, wait for approval, then create the shipment."

The system could generate:

```text
Trigger
   |
Condition
   |
Notification
   |
Approval
   |
Shipment
```

The generated workflow must still pass normal validation and authorization.

---

# 101. Future Enhancement — Marketplace

Allow third parties to publish actions:

```text
Slack
Jira
Salesforce
GitHub
Google Sheets
AWS
Azure
```

Each connector implements:

```java
ActionExecutor
```

with metadata:

```text
name
version
inputSchema
outputSchema
authentication
```

---

# 102. Future Enhancement — Workflow Templates

Templates:

```text
Order Processing
Employee Onboarding
Customer Onboarding
Invoice Processing
Incident Management
Deployment Pipeline
```

Users can clone templates.

---

# 103. Future Enhancement — Human-in-the-Loop

Support:

```text
APPROVAL
REJECTION
FORM_INPUT
MANUAL_REVIEW
```

This transforms the system from simple automation into a business process platform.

---

# 104. Future Enhancement — Workflow Analytics

Dashboard:

```text
Executions
Success Rate
Failure Rate
Average Duration
P95 Duration
Retries
Top Failing Steps
External API Latency
```

Example:

```text
Workflow: Order Processing

Executions       1,240,000
Success            98.7%
Failed              1.3%
Average Duration    2.1 sec
P95                 5.4 sec
```

---

# 105. Future Enhancement — Tenant Isolation

At very large scale:

```text
Shared Control Plane
        |
        +---- Tenant A worker pool
        |
        +---- Tenant B worker pool
        |
        +---- Tenant C worker pool
```

Premium tenants could receive:

```text
Dedicated queues
Dedicated workers
Dedicated databases
```

depending on requirements.

---

# 106. Final Architecture

```text
                         CLIENT
                           |
                           v
                    +-------------+
                    | API Gateway |
                    +------+------+
                           |
                           v
                  +-------------------+
                  | Workflow Service  |
                  +---------+---------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
       +-------------+              +-------------+
       | PostgreSQL  |              | Secret Mgr  |
       +-------------+              +-------------+
              |
              |
              v
       +-------------+
       | Outbox      |
       +------+------+
              |
              v
       +-------------+
       | Message Bus |
       |   Kafka     |
       +------+------+
              |
      +-------+-------+----------------+
      |               |                |
      v               v                v
+-----------+   +-----------+   +-----------+
| Worker 1  |   | Worker 2  |   | Worker N  |
+-----+-----+   +-----+-----+   +-----+-----+
      |               |               |
      +---------------+---------------+
                      |
                      v
              +---------------+
              | Action Layer  |
              +-------+-------+
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       HTTP        Email        SMS
          |           |           |
          +-----------+-----------+
                      |
                      v
              External Systems


Scheduler ---> Message Bus
Event Bus ---> Trigger Service ---> Message Bus
Webhook ----> Trigger Service ---> Message Bus

Workers ---> Execution State ---> PostgreSQL
Workers ---> Metrics/Tracing ---> Observability
Workers ---> Failed Tasks ------> DLQ
```

---

# 107. End-to-End Mental Model

The entire system can be remembered as:

```text
DEFINE
   |
   v
VALIDATE
   |
   v
PUBLISH VERSION
   |
   v
TRIGGER
   |
   v
CREATE EXECUTION
   |
   v
FIND READY NODES
   |
   v
CREATE TASK
   |
   v
QUEUE
   |
   v
WORKER
   |
   v
EXECUTE ACTION
   |
   +-------> RETRY
   |
   +-------> DLQ
   |
   v
PERSIST RESULT
   |
   v
EVALUATE EDGES
   |
   v
NEXT READY NODE
   |
   v
...
   |
   v
WORKFLOW COMPLETED
```

---

# 108. Recommended Interview Flow

When asked this question in an interview, do not immediately start drawing Kafka.

Follow this order:

```text
1. Clarify requirements
        ↓
2. Define workflow model
        ↓
3. Define scale
        ↓
4. Define APIs
        ↓
5. Define data model
        ↓
6. Draw HLD
        ↓
7. Explain execution engine
        ↓
8. Explain queue/workers
        ↓
9. Explain scheduling
        ↓
10. Explain retries
        ↓
11. Explain idempotency
        ↓
12. Explain failure recovery
        ↓
13. Explain consistency
        ↓
14. Explain SOLID/LLD
        ↓
15. Explain scalability
        ↓
16. Discuss bottlenecks
        ↓
17. Future enhancements
```

---

# 109. The Most Important Interview Follow-Ups

If this is being used for senior Java/Spring system-design preparation, focus particularly on these questions:

### Q1

**How does the workflow engine know which node should execute next?**

Expected concepts:

```text
DAG
Graph traversal
Dependencies
Conditions
Ready nodes
State machine
```

### Q2

**What happens when a worker crashes after executing an external API but before updating the database?**

Expected:

```text
At-least-once
Idempotency
Idempotency key
Retry
Deduplication
```

### Q3

**How do you atomically update database state and publish a task?**

Expected:

```text
Transactional Outbox
```

### Q4

**How do you execute workflows that wait for 24 hours?**

Expected:

```text
Persistent timer
WAITING state
No blocked thread
Timer -> Queue
```

### Q5

**How do you handle workflow changes while an execution is running?**

Expected:

```text
Immutable versions
Execution pins version
```

### Q6

**How do you execute branches in parallel?**

Expected:

```text
Dependency graph
Ready-node detection
Concurrent workers
Join condition
```

### Q7

**How would you implement a new action such as Slack without changing the workflow engine?**

Expected:

```text
Strategy
Registry
Open/Closed Principle
Dependency Inversion
```

### Q8

**How do you prevent a single tenant from overwhelming the platform?**

Expected:

```text
Rate limiting
Quotas
Fair scheduling
Partitioning
Dedicated worker pools
```

### Q9

**How do you debug a failed workflow?**

Expected:

```text
Execution ID
Step execution history
Correlation ID
Distributed tracing
Structured logs
Input/output metadata
Retry history
```

### Q10

**How would you scale this from 1,000 executions/day to 100 million executions/day?**

Expected discussion:

```text
Stateless APIs
Partitioned queues
Horizontal workers
Database sharding/partitioning
Event-driven architecture
Caching
Tenant isolation
Backpressure
Autoscaling
Observability
```

---

# 110. Final Design Principles

The most important architectural principles are:

```text
1. Workflow definitions are immutable after publishing.

2. Every execution is tied to a specific workflow version.

3. Execution state is durable.

4. Workers are stateless.

5. Tasks are delivered at least once.

6. Side effects must be idempotent.

7. Long waits must not occupy worker threads.

8. DB + message publishing should use an outbox pattern.

9. New actions should be pluggable.

10. Workflow execution should be modeled as a state machine.

11. Workflow structure should be represented as a graph.

12. Parallelism should come naturally from independent graph branches.

13. Retry policies should be configurable.

14. External dependencies should have rate limiting and circuit breakers.

15. Tenant isolation must be enforced at every layer.

16. Observability is part of the design, not an afterthought.

17. Domain logic should depend on abstractions rather than infrastructure.

18. SOLID principles should guide the LLD.
```

# 111. One-Line Architecture Summary

> **A scalable workflow automation platform is essentially a durable graph-based state machine whose tasks are asynchronously executed by stateless workers, with persistent execution state, versioned workflow definitions, reliable messaging, idempotent side effects, configurable retries, and pluggable action executors.**

This gives you a strong foundation for both the **HLD interview discussion** and the subsequent **Java/Spring LLD implementation**.
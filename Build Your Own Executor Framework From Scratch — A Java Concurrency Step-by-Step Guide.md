# Build Your Own Executor Framework From Scratch — A Java Concurrency Step-by-Step Guide

> **Goal:** Build a working replacement for `java.util.concurrent.ExecutorService`/`ThreadPoolExecutor` — a hand-rolled blocking queue, reusable worker threads, core/max pool sizing with idle timeout, rejection policies, `Future`-returning task submission with a real state machine, and both graceful and immediate shutdown — deriving every design decision from a concrete problem the previous step left unsolved.
>
> This guide assumes the concurrency foundations already built in the companion [ConcurrentHashMap guide](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) (the Java Memory Model, `volatile`, CAS, `synchronized`) and reuses that vocabulary rather than re-deriving it from zero.

---

# 1. What We Are Building

We are building **MiniExecutor** — a task-execution framework architecturally close to the real `ThreadPoolExecutor`. By the end of this guide you will have:

- An **`Executor`/`ExecutorService`**-shaped API that decouples *submitting* work from *how* it eventually runs.
- A **hand-built blocking queue** (`put`/`take`, using `ReentrantLock` + `Condition`, not `wait`/`notify`) as the handoff structure between callers and workers.
- **Reusable worker threads** that loop, pulling tasks from the queue, instead of a thread being created and destroyed per task.
- The real **core/max pool sizing model**: a fixed number of always-alive core threads, additional threads spun up only once the queue is full, and idle non-core threads reaped after a timeout.
- **Rejection policies** (`AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy`, `DiscardOldestPolicy`) for the moment the pool and queue are both genuinely full.
- A real **`FutureTask`** implementation — a state machine, not just a wrapper — supporting `get()`, timeouts, and cancellation.
- Both **graceful (`shutdown()`)** and **immediate (`shutdownNow()`)** termination, and `awaitTermination()`.
- A look at **`ScheduledExecutorService`** (delayed/periodic tasks via a `DelayQueue`), **`ForkJoinPool`**'s different work-stealing architecture, and **virtual threads** as an alternative to pooling.

```text
Caller thread                     MiniThreadPoolExecutor                    Worker threads
     |                                     |                                       |
     |-- execute(task) ------------------->|                                       |
     |                          core threads free? -> hand off directly ---------->| runs task
     |                          core threads busy? -> enqueue -------------------->| take()s it eventually
     |                          queue full, below max? -> spawn a new worker ----->| runs task
     |                          queue full, at max? -> RejectedExecutionHandler    |
     |<-- returns immediately (execute) or a Future (submit) --------------------->|
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain exactly why `new Thread(task).start()` per task is dangerous at scale, in the same concrete terms the [MiniTomcat guide's §15](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already established, and why a pool of reusable threads fixes it.
- Implement a correct blocking queue using `ReentrantLock`/`Condition`, and explain why that's the modern replacement for `wait()`/`notify()`.
- State, from memory, the *exact* order `ThreadPoolExecutor.execute()` makes its decision (core → queue → max → reject) — and explain why that specific order, not a more "obvious" one, is what real `ThreadPoolExecutor` uses.
- Explain why core threads don't time out by default but non-core ones do, and what `allowCoreThreadTimeOut` changes.
- Implement `FutureTask` as a genuine state machine, and explain what `cancel(true)` actually does to a running thread.
- Explain the real difference between `shutdown()` and `shutdownNow()`, and what happens to a task sitting in the queue under each.
- Give a defensible thread-pool size for a stated CPU-bound or I/O-bound workload, from a formula you can derive, not one you memorized.

---

# 3. Why Build This? (Interview Motivation)

> **"Implement a fixed-size thread pool executor supporting task submission with a `Future`-based result, a bounded queue, and graceful shutdown. Explain exactly what happens when a task is submitted and every worker is busy — and what happens when the queue is also full."**

This is one of the most common **senior Java/backend interview questions** because `ExecutorService` is used constantly but understood only shallowly by most engineers — reproducing it forces genuine understanding of:

- **Producer-consumer coordination** — the blocking queue between submitters and workers is the same pattern behind message queues, connection pools, and the [MiniTomcat guide's own request-handling pool](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>).
- **Resource bounding** — exactly the "bound every resource" discipline that guide's §15–§19 already argued for, now built rather than just configured.
- **Asynchronous result handling** — `Future`/`FutureTask` is a small but genuinely tricky state machine, and a common source of subtle bugs (blocking forever on `get()`, mishandling cancellation) when misunderstood.
- **Graceful degradation** — rejection policies and shutdown semantics are about behaving correctly and predictably under overload, not just under the happy path.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | `ReentrantLock`/`Condition`, `AtomicInteger`, and `Thread.ofVirtual()` (§49) are all first-class here. |
| Concurrency primitives | `java.util.concurrent.locks.ReentrantLock`/`Condition`, `java.util.concurrent.atomic.AtomicInteger` | The modern replacement for `synchronized`/`wait`/`notify` — see §13 for why. |
| No `java.util.concurrent.Executors`/`ThreadPoolExecutor` | — | The entire point is to build what those classes are built from, not call them. |
| Testing | JUnit 5 + a concurrent stress harness (same shape as the [ConcurrentHashMap guide's §59](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>)) | Pool-sizing and shutdown bugs are timing-dependent — only real concurrent load exposes them reliably. |

---

# 5. Project Structure

```text
miniexecutor/
├── src/main/java/com/example/miniexecutor/
│   ├── api/
│   │   ├── MiniExecutor.java              // §6
│   │   ├── MiniExecutorService.java       // §8
│   │   ├── MiniFuture.java                // §10
│   │   └── MiniCallable.java
│   ├── queue/
│   │   └── MiniBlockingQueue.java         // §12, §14
│   ├── pool/
│   │   ├── Worker.java                    // §16-§20
│   │   ├── MiniThreadPoolExecutor.java    // §21-§26, §36-§38
│   │   ├── RejectedExecutionHandler.java  // §28
│   │   └── RejectionPolicies.java         // §29
│   ├── future/
│   │   └── MiniFutureTask.java            // §30-§34
│   └── scheduled/
│       ├── DelayQueue.java                // §47
│       └── MiniScheduledExecutor.java     // §46
└── src/test/java/com/example/miniexecutor/
    ├── MiniBlockingQueueTest.java
    ├── PoolSizingTest.java
    ├── RejectionPolicyTest.java
    ├── FutureTaskCancellationTest.java
    └── ShutdownSemanticsTest.java
```

For §42 onward, one addition ties this executor into the MiniSpring companion guide's bean container:

```text
miniexecutor-spring-integration/
└── src/main/java/com/example/miniexecutor/async/
    ├── Async.java                     // §43 — the annotation itself
    ├── AsyncExecutorRegistry.java     // §43
    ├── AsyncInvocationHandler.java    // §45
    └── AsyncAnnotationBeanPostProcessor.java // §46
```

---

# 6. Phase 1 — The Executor Interface: Decoupling "What" from "How"

```java
// api/MiniExecutor.java
public interface MiniExecutor {
    void execute(Runnable task);
}
```

One method. That's deliberate — `Executor` (the real `java.util.concurrent` interface this mirrors) exists purely to separate **submitting** a unit of work from **deciding how it runs** (a new thread? a pooled thread? the calling thread itself? a remote machine?). Code that depends only on `MiniExecutor` never needs to change if the execution strategy behind it changes from, say, a thread pool (§21) to a direct-call stub in a test — the entire point of the interface is what it *doesn't* let a caller see.

---

# 7. Why Executor Exists: The Problem With new Thread() Per Task

Without this abstraction, submitting background work usually means `new Thread(task).start()` directly at the call site — exactly the anti-pattern the [MiniTomcat guide's §14–§15](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already traces in full: unbounded thread creation under load exhausts stack memory and OS scheduling capacity long before CPU becomes the bottleneck, with no single place in the code to apply a limit. `MiniExecutor` is the seam where that limit gets enforced — once — instead of needing to be remembered at every call site that wants to run something in the background.

---

# 8. Phase 2 — ExecutorService: Lifecycle and Submission

```java
// api/MiniExecutorService.java
public interface MiniExecutorService extends MiniExecutor {
    <T> MiniFuture<T> submit(Callable<T> task);   // §9-§10 — a task that PRODUCES a result
    MiniFuture<Void> submit(Runnable task);        // a task that doesn't, wrapped for a uniform return type

    void shutdown();                                // §36 — graceful: finish what's running/queued, then stop
    List<Runnable> shutdownNow();                   // §37 — immediate: stop now, return what never ran
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException; // §38
    boolean isShutdown();
    boolean isTerminated();
}
```

`ExecutorService` extends the bare `Executor` with exactly two new concerns: **getting a result back** (`submit`, returning a `Future`) and **lifecycle management** (`shutdown`/`shutdownNow`/`awaitTermination`) — everything else in this guide is in service of implementing these two concerns correctly.

---

# 9. Runnable vs Callable: Tasks With and Without a Result

```java
@FunctionalInterface
public interface MiniCallable<V> {
    V call() throws Exception; // unlike Runnable.run(), returns a value AND can throw a checked exception
}
```

`Runnable.run()` returns nothing and cannot throw a checked exception — perfectly fine for fire-and-forget work (§47's `logAnalyticsEvent` example later), but useless for a task whose entire purpose is to *compute something*. `Callable<V>` exists specifically to carry that result (and any checked exception the computation might throw) back out — the two interfaces aren't redundant, they're modeling two genuinely different shapes of work.

---

# 10. Phase 3 — The Future Interface: Getting a Result Later

```java
// api/MiniFuture.java
public interface MiniFuture<V> {
    boolean cancel(boolean mayInterruptIfRunning); // §33
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;                       // blocks until the result exists
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException; // §34
}
```

`submit()` (§8) returns one of these **immediately**, before the task has necessarily even started running — `MiniFuture` is a handle to a result that will exist *eventually*, not the result itself. Everything interesting about `Future` is in exactly how `get()` waits for that eventuality correctly, how `cancel()` interacts with a task that might already be mid-execution, and how a checked exception thrown inside the task's `call()` gets faithfully carried across a thread boundary to the caller of `get()` — all three built in §30–§34.

---

# 11. Why a Blocking Queue Is the Right Handoff Structure

Once tasks are submitted faster than workers can run them (or workers are momentarily all busy), submitted tasks need somewhere to wait — a **queue**. It needs to be a specific *kind* of queue, though: an ordinary `java.util.Queue` isn't safe for concurrent access at all, and even a merely thread-safe one doesn't solve the real problem — a worker calling `take()` needs to **block** when the queue is empty (there's nothing productive to do but wait) and automatically wake up the instant something is enqueued, rather than either spinning in a busy loop (wasting CPU) or returning `null` and forcing the worker to poll on a timer (wasting latency). A **blocking queue** is exactly a queue with that wait-and-wake-up behavior built in.

---

# 12. Phase 4 — Building a Minimal BlockingQueue From Scratch

```java
// queue/MiniBlockingQueue.java
public class MiniBlockingQueue<E> {
    private final Object[] items;
    private int head, tail, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition(); // signaled when an item is added
    private final Condition notFull = lock.newCondition();  // signaled when an item is removed

    public MiniBlockingQueue(int capacity) { this.items = new Object[capacity]; }
}
```

A **circular buffer** (`head`/`tail` wrapping around via modulo) backs the queue — a fixed-size array reused indefinitely, rather than a linked structure allocating a new node per element, which avoids per-`put` allocation entirely on the hot path.

---

# 13. wait()/notify() vs Modern Lock/Condition — Why We Use the Latter

Java's original concurrency primitives — `synchronized`, `Object.wait()`/`notify()`/`notifyAll()` — can absolutely implement a blocking queue, but `ReentrantLock`/`Condition` (used throughout this guide) improve on them in ways that matter directly here:

| | `synchronized` + `wait`/`notify` | `ReentrantLock` + `Condition` |
|---|---|---|
| Number of wait-sets per lock | Exactly one — every waiter, regardless of *what* it's waiting for, wakes on any `notify` | **Multiple**, via separate `Condition` objects — `notEmpty` and `notFull` (§12) let a `put()` wake only queue-space-waiters and a `take()` wake only item-waiters, never both indiscriminately |
| Interruptible waiting | `wait()` throws `InterruptedException`, but there's no built-in timed-wait-with-clean-cancellation story beyond `wait(long)`'s coarser semantics | `awaitNanos`/`await(timeout, unit)` give precise, first-class timeout support — directly used by §34's `get(timeout, unit)` |
| Fairness options | None | `ReentrantLock`'s constructor can request FIFO fairness among waiting threads, trading some throughput for predictable ordering |

Using two separate `Condition`s (`notEmpty`, `notFull`) instead of one shared wait-set is the concrete, load-bearing reason this guide reaches for `Lock`/`Condition` rather than `synchronized`/`wait`/`notify` — it avoids waking up threads that have no chance of proceeding (a `put()`-waiter woken by a `take()`, for instance, when only *another* `put()`-waiter could actually make progress), which matters directly for throughput under real contention.

---

# 14. Phase 5 — Implementing put() and take() With ReentrantLock + Condition

```java
public void put(E item) throws InterruptedException {
    lock.lock();
    try {
        while (count == items.length) {      // WHILE, not if — see the note below
            notFull.await();                  // releases the lock, sleeps, re-acquires the lock before returning
        }
        items[tail] = item;
        tail = (tail + 1) % items.length;
        count++;
        notEmpty.signal();                    // wake exactly one waiting take() — there's exactly one new item to give it
    } finally {
        lock.unlock();
    }
}

@SuppressWarnings("unchecked")
public E take() throws InterruptedException {
    lock.lock();
    try {
        while (count == 0) {
            notEmpty.await();
        }
        E item = (E) items[head];
        items[head] = null;                   // avoid holding a stale reference — lets the GC reclaim it
        head = (head + 1) % items.length;
        count--;
        notFull.signal();                     // wake exactly one waiting put() — there's exactly one new slot for it
        return item;
    } finally {
        lock.unlock();
    }
}
```

**Why `while (count == items.length)` and not `if`:** between `notFull.await()` returning and this thread re-acquiring the lock, **another thread could have raced in and refilled the slot that was just freed** — checking the condition again after waking (a "spurious wakeup" guard, but more importantly a genuine re-check against real races) is not defensive paranoia, it's required for correctness; skipping it is one of the most common blocking-queue implementation bugs.

---

# 15. Bounded vs Unbounded Queues and Why Capacity Matters

§12's `MiniBlockingQueue` is deliberately **bounded** — a fixed-size backing array. An **unbounded** queue (no capacity limit at all) might seem simpler, but it silently removes an entire category of backpressure this guide's pool design depends on: with §12's queue, `put()` genuinely **blocks** once the queue is full, which is exactly the signal §23's `execute()` decision logic uses to decide "should I spin up another worker thread, or is the pool truly saturated?" An unbounded queue can never report "full," which — as the [MiniTomcat guide's §19](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already warns — silently defeats a pool's `maximumPoolSize` entirely, since step 3 of §23's decision order (grow past `corePoolSize`) never triggers if the queue always has room.

---

# 16. Phase 6 — The Worker: A Thread That Loops Forever, Pulling Tasks

```java
// pool/Worker.java
public class Worker implements Runnable {
    private final MiniBlockingQueue<Runnable> queue;
    private final MiniThreadPoolExecutor pool;
    volatile Thread thread;
    volatile boolean running = true;

    Worker(MiniBlockingQueue<Runnable> queue, MiniThreadPoolExecutor pool) {
        this.queue = queue;
        this.pool = pool;
    }

    @Override
    public void run() {
        this.thread = Thread.currentThread();
        try {
            while (running) {
                Runnable task = pool.getTask(this); // §25 — may return null on an idle timeout, ending this worker
                if (task == null) break;
                runTask(task); // §19
            }
        } finally {
            pool.workerFinished(this); // deregister — §25, §26
        }
    }
}
```

The entire idea of a thread pool lives in this one loop: a `Worker` is a thread that **never returns from `run()` after just one task** — it keeps asking the pool for the next piece of work until told to stop, which is precisely what makes it reusable rather than a one-shot `Thread`.

---

# 17. Why Workers Reuse Threads Instead of Creating One Per Task

Creating a native OS thread is a genuinely expensive operation (stack allocation, kernel-level bookkeeping) — the [MiniTomcat guide's §15](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already quantifies this cost. A `Worker` that loops (§16) pays that thread-creation cost **once** and amortizes it across every task it ever runs, which is the entire performance argument for pooling threads at all, as opposed to `new Thread(task).start()` per submission — the two approaches run identical task code, but one pays thread-creation overhead per task and the other pays it once per worker's entire lifetime.

---

# 18. Phase 7 — Wiring Workers to the Queue

```java
// MiniThreadPoolExecutor.java (partial — full class assembled across §21-§38)
Runnable getTask(Worker worker) {
    try {
        if (shouldWorkerTimeOut(worker)) {                          // §25 — non-core workers past keepAliveTime
            return queue.poll(keepAliveTime, keepAliveUnit);         // returns null if nothing arrives within the timeout
        }
        return queue.take();                                        // core workers (or timeout disabled) wait forever
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return null; // treat interruption as "stop this worker" — used by shutdownNow(), §37
    }
}
```

`getTask` is the single method that ties a `Worker`'s loop (§16) to both the shared queue (§12–§14) and the pool's own sizing/timeout policy (§25) — a worker never talks to the queue directly, so every sizing decision the pool wants to make can be centralized here rather than scattered across worker instances.

---

# 19. Handling an Exception Thrown Inside a Task

```java
private void runTask(Runnable task) {
    try {
        task.run();
    } catch (RuntimeException | Error e) {
        pool.getUncaughtExceptionHandler().accept(task, e); // logged, NEVER allowed to kill the worker thread
    }
}
```

An uncaught exception inside `task.run()` must **never** be allowed to propagate out of `Worker.run()` — if it did, the exception would terminate that worker's thread entirely, silently shrinking the pool by one every time any submitted task throws, until (given enough failing tasks) the pool has no workers left at all. Catching broadly here and routing to a configurable handler is what keeps one bad task from degrading the entire pool's capacity.

---

# 20. Worker Thread Naming and ThreadFactory

```java
public interface MiniThreadFactory {
    Thread newThread(Runnable r);
}

public class NamedThreadFactory implements MiniThreadFactory {
    private final String prefix;
    private final AtomicInteger counter = new AtomicInteger(1);
    public NamedThreadFactory(String prefix) { this.prefix = prefix; }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setDaemon(false); // non-daemon — matches the MiniTomcat guide's §16 reasoning: let in-flight work finish before JVM exit
        return t;
    }
}
```

Giving every worker thread a distinguishable name (`"miniexecutor-worker-3"` rather than the JVM default `"Thread-17"`) is not cosmetic — it's what makes a thread dump under production load actually diagnosable, letting you see at a glance how many pool threads exist, which pool they belong to, and (cross-referenced with their stack traces) what each one is currently blocked on.

---

# 21. Phase 8 — Designing MiniThreadPoolExecutor's Fields

```java
// pool/MiniThreadPoolExecutor.java
public class MiniThreadPoolExecutor implements MiniExecutorService {
    private final int corePoolSize;
    private final int maximumPoolSize;
    private final long keepAliveTime;
    private final TimeUnit keepAliveUnit;
    private final MiniBlockingQueue<Runnable> workQueue;
    private final MiniThreadFactory threadFactory;
    private final RejectedExecutionHandler rejectionHandler; // §28

    private final Set<Worker> workers = ConcurrentHashMap.newKeySet(); // every currently-live worker
    private final AtomicInteger poolSize = new AtomicInteger(0);        // current worker COUNT — read far more than it's written
    private volatile boolean shutdown = false;

    public MiniThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit keepAliveUnit,
                                   MiniBlockingQueue<Runnable> workQueue, MiniThreadFactory threadFactory,
                                   RejectedExecutionHandler rejectionHandler) {
        if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize) {
            throw new IllegalArgumentException("Invalid pool size configuration");
        }
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.keepAliveTime = keepAliveTime;
        this.keepAliveUnit = keepAliveUnit;
        this.workQueue = workQueue;
        this.threadFactory = threadFactory;
        this.rejectionHandler = rejectionHandler;
    }
}
```

Every one of these seven fields corresponds to a distinct, real decision this guide's §23 decision logic makes — there is no field here that isn't load-bearing for a specific piece of `execute()`'s behavior.

---

# 22. Phase 9 — Implementing execute(Runnable)

```java
@Override
public void execute(Runnable task) {
    if (shutdown) {
        rejectionHandler.rejectedExecution(task, this); // §28-§29 — reject immediately, don't even try to enqueue
        return;
    }
    if (poolSize.get() < corePoolSize) {
        if (addWorker(task, true)) return;                  // §24 — "true" marks this as a CORE worker (§26)
    }
    if (workQueue.offer(task)) {                              // non-blocking enqueue attempt — see §14's put(), but non-blocking here
        return;
    }
    if (poolSize.get() < maximumPoolSize) {
        if (addWorker(task, false)) return;                  // "false" marks this as a NON-core worker (§25, §26)
    }
    rejectionHandler.rejectedExecution(task, this);           // pool AND queue are both genuinely full — §27-§29
}
```

`workQueue.offer(task)` (a non-blocking "try to enqueue, return `false` if full" variant of §14's blocking `put()`) is what lets `execute()` make an immediate decision rather than ever blocking the *caller* — a design choice worth naming explicitly, since a caller invoking `execute()` reasonably expects it to return quickly, delegating the actual waiting to whichever `Worker` eventually calls `take()`, not to the submitting thread itself.

---

# 23. The Exact Decision Order: Core, Then Queue, Then Max, Then Reject (Deriving It)

§22's four-step order is not arbitrary — each step is checked only because the previous one specifically failed, and the order itself encodes a real preference:

1. **Prefer spinning up a core thread first**, even if an existing core thread happens to be idle at that exact instant — real `ThreadPoolExecutor` does this too. This sounds wasteful, but it guarantees that a burst of `corePoolSize` tasks arriving together gets `corePoolSize` threads working on them **immediately**, rather than some sitting in the queue behind others purely due to scheduling luck.
2. **Only once core capacity is fully committed, try the queue** — this is the pool's actual buffering mechanism, and it's tried *before* growing the pool further, specifically because a queued task waiting briefly for an existing thread to free up is normally cheaper than paying to create a whole new thread.
3. **Only once the queue itself is full does the pool grow past `corePoolSize`** — this is the step [the MiniTomcat guide's §17](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already flags as surprising: the pool does **not** grow just because tasks are queueing, only once queueing capacity is *also* exhausted. A bounded queue (§15) is what makes this step reachable at all.
4. **Only once every option above is exhausted — core threads busy, queue full, and already at `maximumPoolSize` — does rejection (§27) happen.**

Getting this order backwards (e.g., trying to grow the pool before trying the queue) would mean a bounded queue's entire purpose — absorbing a short burst without needing new threads — never actually gets used, defeating the reason a queue is in this design at all.

---

# 24. Phase 10 — Growing Beyond corePoolSize: Non-Core Threads

```java
private boolean addWorker(Runnable firstTask, boolean isCore) {
    int currentSize = poolSize.get();
    int limit = isCore ? corePoolSize : maximumPoolSize;
    if (currentSize >= limit) return false;
    if (!poolSize.compareAndSet(currentSize, currentSize + 1)) {
        return addWorker(firstTask, isCore); // another thread changed poolSize concurrently — retry with a fresh read
    }

    Worker worker = new Worker(workQueue, this);
    workers.add(worker);
    Thread thread = threadFactory.newThread(worker);
    worker.thread = thread;
    if (firstTask != null) workQueue.offer(firstTask); // hand the triggering task in via the queue — the new worker picks it up immediately
    thread.start();
    return true;
}
```

A **non-core** worker is, mechanically, identical in every way to a core one (same `Worker` class, same `run()` loop) — the *only* difference is which limit (`corePoolSize` vs `maximumPoolSize`) governed whether it was allowed to be created, and, per §25, whether it's willing to sit idle forever waiting for the next task or will eventually time out.

---

# 25. Phase 11 — Idle Timeout: Killing Non-Core Threads With keepAliveTime

```java
// Referenced from Worker.getTask() (§18)
boolean shouldWorkerTimeOut(Worker worker) {
    return isNonCoreWorker(worker) || allowCoreThreadTimeOut; // §26
}

void workerFinished(Worker worker) {
    workers.remove(worker);
    poolSize.decrementAndGet(); // the thread is genuinely gone — the pool can spin up a fresh one later if load returns
}
```

When `getTask` (§18) uses `queue.poll(keepAliveTime, unit)` instead of the blocking `queue.take()`, it returns `null` if no task arrives within that window — `Worker.run()`'s loop (§16) treats a `null` task as "time to stop," and the thread exits, calling `workerFinished` on its way out. This is what keeps a burst-driven pool from permanently holding onto threads it only needed briefly: extra capacity created to absorb a spike (§24) quietly gives itself back once the spike passes and the queue goes idle for longer than `keepAliveTime`.

---

# 26. Why corePoolSize Threads Don't Time Out by Default

Core threads exist specifically to be **always ready** — that's the entire distinction §23's step 1 relies on (prefer a core thread immediately, without even checking the queue first). If core threads timed out and disappeared during a quiet period, the *next* burst of tasks would pay full thread-creation cost all over again for what's supposed to be the pool's baseline, defeating the reason a "core" tier exists separately from the elastic "max" tier at all. `allowCoreThreadTimeOut` (referenced in §25) is an explicit opt-in for the (less common) case where even the baseline threads should be reclaimed during genuinely extended idle periods — a deliberate override of the default, not the default itself.

---

# 27. What Happens When the Pool and Queue Are Both Full

§22's final `if` branch — `poolSize.get() < maximumPoolSize` failing — means every option this design offers has been exhausted: every thread up to the maximum is busy, and the bounded queue (§15) has no room for one more waiting task. Something has to give, and **silently dropping the task** is never an acceptable default (a dropped task looks, from the caller's perspective, identical to one that's just running slowly — until it never completes) — the pool needs to make an explicit, pluggable decision about what happens next.

---

# 28. Phase 12 — The RejectedExecutionHandler Interface

```java
// pool/RejectedExecutionHandler.java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable task, MiniThreadPoolExecutor executor);
}
```

Making rejection a pluggable **strategy** (rather than one hard-coded behavior) is deliberate: "the pool is saturated" is a business-meaningful event whose correct response genuinely differs by workload — a batch job might want to block the submitter until room frees up; a live HTTP request handler (exactly [the MiniTomcat guide's own use of this pattern](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>), §19) might want to fail fast with a `503`. One interface, several interchangeable implementations (§29), lets the same `MiniThreadPoolExecutor` serve both needs without any change to its own code.

---

# 29. Implementing the Four Standard Rejection Policies

```java
public class AbortPolicy implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable task, MiniThreadPoolExecutor executor) {
        throw new RejectedExecutionException("Task " + task + " rejected — pool and queue both full");
    }
}

public class CallerRunsPolicy implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable task, MiniThreadPoolExecutor executor) {
        if (!executor.isShutdown()) task.run(); // runs SYNCHRONOUSLY on the submitting thread — a natural, self-throttling backpressure
    }
}

public class DiscardPolicy implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable task, MiniThreadPoolExecutor executor) { /* deliberately does nothing */ }
}

public class DiscardOldestPolicy implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable task, MiniThreadPoolExecutor executor) {
        executor.getWorkQueue().pollOldest();  // makes room by dropping the LONGEST-WAITING queued task
        executor.execute(task);                 // then retries the newly-submitted one
    }
}
```

`CallerRunsPolicy` deserves the same call-out [the MiniTomcat guide's §19](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) gives it: running the rejected task on the **calling** thread means that thread can't submit anything else until this task finishes — a simple, effective, self-limiting brake on the rate new work arrives, without needing any separate rate-limiting mechanism at all.

---

# 30. Phase 13 — Wrapping a Callable in a FutureTask

```java
// future/MiniFutureTask.java
public class MiniFutureTask<V> implements Runnable, MiniFuture<V> {
    private final Callable<V> task;
    private volatile State state = State.NEW;
    private V result;
    private Throwable exception;
    private volatile Thread runnerThread; // the worker thread actually executing call() — needed for cancel(true), §33

    private enum State { NEW, RUNNING, COMPLETED, FAILED, CANCELLED }

    public MiniFutureTask(Callable<V> task) { this.task = task; }
}
```

`MiniFutureTask` implements **both** `Runnable` (so it can be handed straight to `execute()`, §22) **and** `MiniFuture<V>` (so it can be handed straight back to whoever called `submit()`) — one object plays both roles, which is exactly what lets `submit()` (§32) be a thin wrapper: construct one of these, `execute()` it, return it.

---

# 31. Phase 14 — Implementing FutureTask's State Machine

```java
private final Object lock = new Object();

@Override
public void run() {
    synchronized (lock) {
        if (state != State.NEW) return; // already cancelled before it even started — see §33
        state = State.RUNNING;
        runnerThread = Thread.currentThread();
    }
    try {
        V value = task.call();
        synchronized (lock) {
            if (state == State.RUNNING) { result = value; state = State.COMPLETED; }
            lock.notifyAll(); // wake every thread blocked in get() (§34)
        }
    } catch (Throwable t) {
        synchronized (lock) {
            if (state == State.RUNNING) { exception = t; state = State.FAILED; }
            lock.notifyAll();
        }
    } finally {
        synchronized (lock) { runnerThread = null; }
    }
}
```

The state machine (`NEW → RUNNING → COMPLETED`/`FAILED`, or `NEW`/`RUNNING → CANCELLED`) is what makes concurrent access from multiple directions — the worker thread running `run()`, a caller thread calling `get()`, another thread calling `cancel()` — resolve unambiguously: every transition checks the *current* state before acting, so a cancellation that arrives after the task already completed is a correct no-op, not a race that corrupts an already-delivered result.

---

# 32. Phase 15 — submit() Returning a Future

```java
// MiniThreadPoolExecutor — implementing MiniExecutorService's submit()
@Override
public <T> MiniFuture<T> submit(Callable<T> task) {
    MiniFutureTask<T> futureTask = new MiniFutureTask<>(task);
    execute(futureTask); // MiniFutureTask IS a Runnable (§30) — flows through the exact same execute() logic as §22
    return futureTask;    // the caller gets a handle back immediately, before the task has necessarily even started
}

@Override
public MiniFuture<Void> submit(Runnable task) {
    return submit(() -> { task.run(); return null; }); // adapt a Runnable into a Callable<Void> — reuses everything above
}
```

Nothing about §21–§29's pool logic needed to change to support `submit()` — a `MiniFutureTask` is, from `execute()`'s point of view, just another `Runnable`, which is precisely the payoff of §30's design decision to have it implement both interfaces at once.

---

# 33. Cancellation: What cancel(true) Actually Does

```java
@Override
public boolean cancel(boolean mayInterruptIfRunning) {
    synchronized (lock) {
        if (state != State.NEW && state != State.RUNNING) return false; // already finished — too late to cancel
        if (state == State.RUNNING) {
            if (!mayInterruptIfRunning) return false; // caller explicitly said "don't interrupt a running task"
            if (runnerThread != null) runnerThread.interrupt(); // §18's Worker sees this as InterruptedException
        }
        state = State.CANCELLED;
        lock.notifyAll();
        return true;
    }
}
```

`cancel(true)` does **not** forcibly kill a thread — Java has no safe mechanism to do that. It calls `Thread.interrupt()`, which sets an interrupt flag the running task's own code must **choose to check** (via `Thread.interrupted()`, or implicitly by calling a blocking method like `Thread.sleep()` or `queue.take()`, which throw `InterruptedException` when interrupted). A task that never checks for interruption, and contains no interruptible blocking calls, **will not actually stop** just because `cancel(true)` was called — this is a real, honest limitation worth stating plainly rather than implying cancellation is more forceful than it actually is.

---

# 34. Handling get() Timeouts and Exceptions

```java
@Override
public V get() throws InterruptedException, ExecutionException {
    synchronized (lock) {
        while (state == State.NEW || state == State.RUNNING) lock.wait(); // blocks until run() calls notifyAll()
        return resultOrThrow();
    }
}

@Override
public V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    long deadline = System.nanoTime() + unit.toNanos(timeout);
    synchronized (lock) {
        while (state == State.NEW || state == State.RUNNING) {
            long remainingNanos = deadline - System.nanoTime();
            if (remainingNanos <= 0) throw new TimeoutException();
            lock.wait(remainingNanos / 1_000_000, (int) (remainingNanos % 1_000_000)); // millis + nanos overload
        }
        return resultOrThrow();
    }
}

private V resultOrThrow() throws ExecutionException {
    if (state == State.CANCELLED) throw new CancellationException();
    if (state == State.FAILED) throw new ExecutionException(exception); // wraps the task's own exception, doesn't lose it
    return result;
}
```

Wrapping the task's thrown exception inside `ExecutionException` (rather than rethrowing it directly) is a deliberate, important design choice: it lets `get()`'s caller distinguish, syntactically, between "the task itself failed" (unwrap `ExecutionException.getCause()`) and "something went wrong with `get()`/the Future mechanism itself" (`InterruptedException`, `TimeoutException`, `CancellationException`) — conflating these would make correct error handling by callers considerably harder.

---

# 35. Why Shutdown Needs Two Modes

Stopping a pool has two genuinely different, both legitimate, meanings: **"stop accepting new work, but let everything already accepted finish"** (a normal, graceful deploy/restart) and **"stop everything, right now, and tell me what didn't get to run"** (an emergency shutdown, or a test harness that needs a clean, immediate teardown). Conflating these into one `stop()` method would force every caller into whichever behavior happened to be implemented, even when the other one is what the situation actually calls for.

---

# 36. Phase 16 — Implementing shutdown()

```java
@Override
public void shutdown() {
    shutdown = true; // execute() (§22) now rejects every NEW submission immediately
    // Deliberately does NOT touch the queue or interrupt any worker — every already-queued or already-running
    // task is left completely alone, exactly per the "graceful" contract this method promises.
}
```

The smallest possible implementation is also the correct one here: once `shutdown` is `true`, §22's very first check rejects all new work, while every worker (§16) keeps looping — pulling from the queue via `getTask` (§18) — until the queue is genuinely empty, at which point each worker's `queue.poll`/`take` eventually returns nothing new to do and the workers exit naturally (§25's timeout path, or a dedicated "queue is empty and shutdown is set" check added to `Worker.run()`).

---

# 37. Phase 17 — Implementing shutdownNow()

```java
@Override
public List<Runnable> shutdownNow() {
    shutdown = true;
    List<Runnable> neverRan = workQueue.drainAll(); // §12's queue — every task still waiting, removed and returned
    for (Worker worker : workers) {
        if (worker.thread != null) worker.thread.interrupt(); // §33's exact interruption mechanism, applied pool-wide
    }
    return neverRan; // the caller can inspect/reschedule/log exactly what got abandoned
}
```

`shutdownNow()` makes the opposite tradeoff from `shutdown()` deliberately: it **interrupts every running worker immediately** (subject to the same "the task must actually check for interruption" caveat §33 already names) and **hands back every task that was still waiting**, so nothing simply vanishes without the caller knowing — abrupt, but honest about exactly what didn't get to run.

---

# 38. Phase 18 — awaitTermination()

```java
@Override
public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
    long deadline = System.nanoTime() + unit.toNanos(timeout);
    while (!isTerminated()) {
        if (System.nanoTime() >= deadline) return false; // timed out — some workers are still running
        Thread.sleep(10); // simple polling; a production-grade version would use a Condition signaled by workerFinished (§25)
    }
    return true;
}

@Override
public boolean isTerminated() {
    return shutdown && workers.isEmpty(); // shutdown was requested AND every worker has actually exited
}
```

`awaitTermination` is what lets calling code **wait, with a bound**, for a graceful shutdown to actually finish — the common, correct pattern (`shutdown()` immediately followed by `awaitTermination(...)`, falling back to `shutdownNow()` if the timeout expires) mirrors exactly the three-step graceful shutdown the [MiniTomcat guide's §21](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already established: stop intake, drain with a deadline, then force.

---

# 39. Full Worked Example: Submitting Tasks and Collecting Results

```java
MiniThreadPoolExecutor pool = new MiniThreadPoolExecutor(
    4, 8, 30, TimeUnit.SECONDS,
    new MiniBlockingQueue<>(100),
    new NamedThreadFactory("report-worker"),
    new CallerRunsPolicy()
);

List<MiniFuture<Integer>> futures = new ArrayList<>();
for (int i = 0; i < 20; i++) {
    int input = i;
    futures.add(pool.submit(() -> expensiveComputation(input))); // returns immediately, 20 times
}

int total = 0;
for (MiniFuture<Integer> future : futures) {
    total += future.get(); // blocks only as long as THIS particular task hasn't finished yet
}

pool.shutdown();
pool.awaitTermination(10, TimeUnit.SECONDS);
```

20 tasks submitted essentially instantly; at most 4 run concurrently at first (`corePoolSize`), with the rest queueing (§23's step 2) and, if the queue fills, spilling into up to 4 more threads (`maximumPoolSize`, step 3) — all invisible to this calling code, which only ever sees "submit, get a `Future`, call `get()` when the result is needed."

---

# 40. Tracing a Task End to End

```text
pool.submit(callable)
  -> new MiniFutureTask(callable)                          (§30)
  -> pool.execute(futureTask)                               (§22)
       poolSize < corePoolSize? -> addWorker(futureTask, isCore=true) -> new Worker, new Thread, thread.start() (§24)
  -> Worker.run() starts on its own thread                  (§16)
       getTask(this) -> queue.take() or the task handed in directly via addWorker's offer()  (§18)
       runTask(task) -> task.run()                           (§19, and §31's FutureTask.run())
            state: NEW -> RUNNING -> COMPLETED (or FAILED)   (§31)
            lock.notifyAll()                                 -- wakes any thread blocked in future.get()
  -> caller's future.get() unblocks, returns the result       (§34)
  -> Worker loops back to getTask() for the NEXT task, reusing the SAME thread (§16-§17)
```

Every mechanism built in §6–§38 appears somewhere in this one trace — a single submitted task exercises the full pool sizing decision, the worker loop, the future state machine, and (implicitly) the queue's `put`/`take` machinery underneath it all.

---

# 41. Final Architecture (of the Core Executor)

```text
                        Caller
                          |
                execute(task) / submit(callable)
                          |
                          v
              +------------------------+
              | MiniThreadPoolExecutor  |
              |  (§21-§29)               |
              +-----------+------------+
                          |
        +------------------+------------------+
        v                                     v
  poolSize < core?                    queue.offer() succeeded?
  addWorker(core=true) (§24)               |  no -> poolSize < max?
        |                                   |        addWorker(core=false) (§24)
        v                                   v              |
  +------------------------+      +------------------+     v
  |   Worker thread(s)      |<---->| MiniBlockingQueue |   (still full) -> RejectedExecutionHandler (§28-§29)
  |  loop: getTask()->run()  |      |  put()/take() (§12-14)|
  |  (§16-§20, §25-§26)      |      +------------------+
  +-----------+--------------+
              |
              v
      MiniFutureTask.run()
      state machine (§30-§31)
              |
              v
      caller's future.get() (§32, §34)
      cancel() (§33)

  shutdown()/shutdownNow()/awaitTermination() (§35-§38) control the whole lifecycle from outside this loop
```

---

# 42. Why a Declarative @Async Is Worth Building on Top of MiniExecutor

Everything through §41 requires a caller to explicitly write `pool.submit(() -> doWork())` — correct, but it means "this method runs asynchronously" is a decision made **at every call site**, easy to forget, and impossible to change centrally later (want `sendEmail` to suddenly run synchronously in tests? every call site needs editing). Spring's real `@Async` solves this by making "runs asynchronously" a property of the **method's declaration** instead — annotate it once, and every caller gets asynchronous behavior automatically, without their own code changing at all. Building this requires solving a problem this guide hasn't touched yet: **how do you make an ordinary method call run through the executor, without editing the method's own body or every caller?**

---

# 43. Phase 19 — The @Async Annotation and a Named-Executor Registry

```java
// async/Async.java
@Retention(RetentionPolicy.RUNTIME) // MUST be RUNTIME — reflection needs to see it long after compilation, exactly
@Target(ElementType.METHOD)          // like every annotation in the MiniSpring companion guide's §8
public @interface Async {
    String value() default "default"; // which registered executor should run this method — configurable per method
}
```

```java
// async/AsyncExecutorRegistry.java
public class AsyncExecutorRegistry {
    private final Map<String, MiniExecutorService> executors = new ConcurrentHashMap<>();

    public void register(String name, MiniExecutorService executor) { executors.put(name, executor); }

    public MiniExecutorService get(String name) {
        MiniExecutorService executor = executors.get(name);
        if (executor == null) throw new IllegalStateException("No @Async executor registered under name: " + name);
        return executor;
    }
}
```

```java
// A developer configures this once, at startup, choosing pool shape PER named executor
@Component
public class AsyncConfig {
    @PostConstruct
    public void configureExecutors(AsyncExecutorRegistry registry) {
        registry.register("default", new MiniThreadPoolExecutor(4, 8, 30, TimeUnit.SECONDS,
            new MiniBlockingQueue<>(100), new NamedThreadFactory("async-default"), new CallerRunsPolicy()));
        registry.register("email-sender", new MiniThreadPoolExecutor(2, 4, 60, TimeUnit.SECONDS,
            new MiniBlockingQueue<>(50), new NamedThreadFactory("async-email"), new AbortPolicy()));
    }
}
```

This is the entire *configuration* half of the feature — a name maps to a fully-tunable `MiniThreadPoolExecutor` (§21), and `@Async("email-sender")` on a method will later mean "run this on that specific, independently-sized pool," isolating a slow downstream dependency (an SMTP server) from starving unrelated async work.

---

# 44. Understanding Dynamic Proxies: How java.lang.reflect.Proxy Actually Works

The remaining problem — making a call to `notificationService.sendEmail(...)` actually route through `AsyncExecutorRegistry` instead of running `sendEmail`'s body directly on the caller's thread — cannot be solved by annotation processing alone; annotations are just metadata, inert until something reads and acts on them. The mechanism that acts on them here is a **dynamic proxy**, and it's worth understanding in real detail before using it, since it looks close to magic otherwise.

**What `Proxy.newProxyInstance` actually does:** at *runtime* (not compile time), the JVM generates a brand-new class — one that doesn't exist anywhere in your source code or your compiled `.class` files — which implements whatever interface(s) you specify. Every method that interface declares is given a trivial implementation in this generated class: **package up the method that was called and its arguments, and forward the whole thing to a single object you provide**, called the `InvocationHandler`. Nothing about the interface's *real* implementation is touched, copied, or modified — the generated proxy class is a completely separate object that merely *also* implements the same interface, and stands in front of the real one.

```java
Object proxy = Proxy.newProxyInstance(
    classLoader,                    // which ClassLoader defines the generated proxy class
    new Class<?>[]{ NotificationService.class }, // which interface(s) the generated class must implement
    invocationHandler                // WHERE every single method call gets forwarded to
);
```

The three arguments map directly onto the three questions a dynamic proxy has to answer: *where does the generated class live* (the class loader), *what does it pretend to be* (the interfaces), and *who actually decides what happens when one of its methods is called* (the handler). `NotificationService notificationService = (NotificationService) proxy;` then lets calling code use the proxy exactly as if it were a real `NotificationService` — Java's type system has no way to tell the difference from the caller's side, by design.

```java
public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

Every single method call on the proxy — regardless of which interface method was invoked, with whatever arguments — funnels through this **one** `invoke` method. `method` (a `java.lang.reflect.Method` object) tells the handler exactly *which* method was called — this is precisely how §47's `AsyncInvocationHandler` can inspect `method` for an `@Async` annotation and decide, per call, whether to run it synchronously or hand it to an executor.

---

# 45. Phase 20 — Generating a Proxy Instance and Routing Every Call Through One Handler

```java
public class ProxyDemo {
    public static void main(String[] args) {
        NotificationService realService = new NotificationServiceImpl();

        InvocationHandler handler = (proxy, method, methodArgs) -> {
            System.out.println("Intercepted call to: " + method.getName());
            return method.invoke(realService, methodArgs); // forward to the REAL object — pass-through, for now
        };

        NotificationService proxied = (NotificationService) Proxy.newProxyInstance(
            NotificationService.class.getClassLoader(),
            new Class<?>[]{ NotificationService.class },
            handler
        );

        proxied.sendWelcomeEmail("alice@example.com"); // prints "Intercepted call to: sendWelcomeEmail", THEN runs the real method
    }
}
```

This is the entire mechanical foundation everything from §47 onward builds on: a proxy that does nothing more than log, then delegate — the exact same shape as an `@Async`-aware proxy, minus the actual asynchronous dispatch decision. Understanding that `invoke()` is a single, universal interception point for *every* method on the interface — not something wired up per method — is what makes it possible to support `@Async` on any number of methods, on any number of interfaces, with the exact same handful of lines of handler code.

---

# 46. JDK Dynamic Proxies vs Subclassing (CGLIB/ByteBuddy): Why @Async Needs an Interface

`java.lang.reflect.Proxy`'s generated class implements one or more **interfaces** — it fundamentally cannot proxy a plain class that implements no interface, because there is no interface contract for the generated class to implement instead. Real Spring works around this limitation for class-based beans using a **different** technique entirely: libraries like CGLIB or ByteBuddy generate a **subclass** of the target class at runtime, override its methods, and route those overrides through an interceptor — mechanically different (subclassing vs. interface implementation) but conceptually identical in purpose (get *some* generated object between the caller and the real one). This guide's `@Async` implementation (§47–§48) deliberately uses only `java.lang.reflect.Proxy`, which means **it only works on beans that implement an interface** — an honest, explicitly named limitation (§52), not a hidden gap, and precisely why the worked example in §51 defines `NotificationService` as an interface with a separate `NotificationServiceImpl`.

---

# 47. Phase 21 — An AsyncInvocationHandler: Intercepting the Method Call

```java
// async/AsyncInvocationHandler.java
public class AsyncInvocationHandler implements InvocationHandler {
    private final Object target;                        // the REAL bean instance, e.g. a NotificationServiceImpl
    private final AsyncExecutorRegistry executorRegistry;

    public AsyncInvocationHandler(Object target, AsyncExecutorRegistry executorRegistry) {
        this.target = target;
        this.executorRegistry = executorRegistry;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Async async = method.getAnnotation(Async.class); // §44's `method` parameter, read for @Async metadata
        if (async == null) {
            return method.invoke(target, args); // NOT async — pass straight through, synchronous, unchanged behavior
        }

        MiniExecutorService executor = executorRegistry.get(async.value()); // §43 — named-pool lookup

        if (method.getReturnType() == void.class) {
            executor.execute(() -> {
                try { method.invoke(target, args); }
                catch (Throwable t) { logUncaughtAsyncFailure(method, t); } // fire-and-forget — caller never sees this
            });
            return null; // the caller's call to the void async method returns immediately, unconditionally
        }

        if (MiniFuture.class.isAssignableFrom(method.getReturnType())) {
            return executor.submit(() -> method.invoke(target, args)); // §32 — returns a real MiniFuture to the caller
        }

        throw new IllegalStateException("@Async method must return void or a MiniFuture<T>: " + method);
    }
}
```

Reading `method.getAnnotation(Async.class)` **inside `invoke`**, on every call, rather than deciding once at proxy-creation time, is what lets a single proxy correctly handle a mix of `@Async` and ordinary synchronous methods on the same interface — the decision is made fresh, per call, based on exactly which method this particular invocation happens to be.

---

# 48. Phase 22 — Wiring the Proxy Into Bean Creation (a BeanPostProcessor)

```java
// async/AsyncAnnotationBeanPostProcessor.java
public class AsyncAnnotationBeanPostProcessor implements BeanPostProcessor { // the MiniSpring companion guide's §36 extension point
    private final AsyncExecutorRegistry executorRegistry;

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (!hasAnyAsyncMethod(bean.getClass())) return bean; // no @Async anywhere -> return the REAL bean, unproxied

        Class<?>[] interfaces = bean.getClass().getInterfaces();
        if (interfaces.length == 0) {
            throw new IllegalStateException(
                "@Async on " + beanName + " requires it to implement an interface — see §46");
        }
        return Proxy.newProxyInstance(                                     // §44-§45, applied to a REAL MiniSpring bean
            bean.getClass().getClassLoader(),
            interfaces,
            new AsyncInvocationHandler(bean, executorRegistry)
        );
    }

    private boolean hasAnyAsyncMethod(Class<?> clazz) {
        for (Method method : clazz.getMethods()) {
            if (method.isAnnotationPresent(Async.class)) return true;
        }
        return false;
    }
}
```

This is the piece that makes `@Async` feel automatic rather than manual: [MiniSpring's `ApplicationContext`](MiniSpring-Step-by-Step-Guide.md) already runs every registered `BeanPostProcessor` after a bean is constructed and wired, **replacing** the bean reference held in the container with whatever the processor returns (§36-area of that guide's bean-lifecycle phase). Once `postProcessAfterInitialization` returns a proxy instead of the real `NotificationServiceImpl`, **every other bean that gets `NotificationService` injected via `@Autowired` receives the proxy** — they call methods on it exactly as before, with no code of their own aware that anything changed.

---

# 49. Handling void vs Future-Returning @Async Methods Differently

§47's `invoke` branches on the method's declared return type for a concrete reason tied directly back to §10's `MiniFuture` design: a `void @Async` method is inherently **fire-and-forget** — there is no return value for the caller to ever inspect, so the only sensible behavior is to submit the work and return control to the caller immediately, with any exception the task throws captured only in a log, never surfaced to a caller who has nothing to receive it. A method declared to return `MiniFuture<T>`, by contrast, is explicitly telling its callers "I intend to give you a handle to a result eventually" — `executor.submit(...)` (§32) is the exact mechanism that promise requires, and returning anything other than the `MiniFuture` `submit` produces would silently break that contract.

---

# 50. Configuring a Per-Method Executor: @Async("name")

```java
public interface NotificationService {
    @Async("email-sender") // routes to the SMTP-isolated pool configured in §43 — never competes with other async work
    MiniFuture<Void> sendWelcomeEmail(String to);

    @Async // no argument -> uses "default", per @Async's default value (§43)
    void logAnalyticsEvent(String eventName);
}
```

Because §47 reads `async.value()` fresh on every call, two different `@Async`-annotated methods on the **same** interface can route to **two entirely different, independently-sized thread pools** — exactly the isolation [the MiniTomcat guide's thread-pool tuning discussion](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) argues for at the transport layer, now available at the method-annotation level: a slow, rarely-called email pool never steals capacity from a hot, frequently-called analytics-logging pool, because they were never sharing one pool in the first place.

---

# 51. Phase 23 — A Full Worked Example: An Async Notification Service

```java
public interface NotificationService {
    @Async("email-sender")
    MiniFuture<Void> sendWelcomeEmail(String to);

    @Async
    void logAnalyticsEvent(String eventName);
}

@Component
public class NotificationServiceImpl implements NotificationService {
    @Override
    public MiniFuture<Void> sendWelcomeEmail(String to) {
        smtpClient.send(to, "Welcome!"); // a genuinely slow call — exactly why this needed its own pool (§50)
        return MiniFuture.completed(null);
    }

    @Override
    public void logAnalyticsEvent(String eventName) {
        analyticsClient.record(eventName);
    }
}

// Elsewhere, injected normally — completely unaware a proxy is involved
@Service
public class SignupService {
    private final NotificationService notificationService; // this is ACTUALLY the proxy from §48, injected transparently

    @Autowired
    public SignupService(NotificationService notificationService) { this.notificationService = notificationService; }

    public void handleSignup(String email) {
        createAccount(email);
        notificationService.sendWelcomeEmail(email);   // returns INSTANTLY — the real send happens on "email-sender"
        notificationService.logAnalyticsEvent("signup"); // returns INSTANTLY — the real log happens on "default"
        // handleSignup() itself returns to ITS caller without ever waiting on either background call
    }
}
```

`SignupService` never imports `MiniThreadPoolExecutor`, never calls `submit()`, and never knows a proxy stands between it and `NotificationServiceImpl` — every piece of asynchrony this method exhibits was declared entirely on `NotificationService`'s own interface (§50), exactly the "declare it once, benefit everywhere" goal §42 set out to achieve.

---

# 52. Limitations of This Design (Self-Invocation, Interface-Only Proxying) and How Real Spring Handles Them

Two real, worth-naming-honestly gaps, both inherent to the proxy-based approach itself, not bugs in this guide's specific code:

- **Self-invocation bypasses the proxy entirely.** If `NotificationServiceImpl.sendWelcomeEmail` internally called `this.logAnalyticsEvent(...)` directly, that call goes straight to the real object's own method — `this` inside `NotificationServiceImpl` is never the proxy, only external callers holding the injected reference ever go through it. This is a well-documented real Spring `@Async` gotcha for exactly this reason, not something unique to this guide's simplified version.
- **Interface-only proxying (§46).** Any bean that doesn't implement an interface simply cannot use this mechanism at all — real Spring's default configuration silently falls back to CGLIB subclassing specifically to avoid this restriction, at the cost of needing a non-`final` class with a non-`final` method and a byte-code-generation dependency this guide deliberately didn't take on.

Naming both limitations explicitly, rather than letting a user discover them as confusing runtime surprises, is the same "honest gap" discipline the [TinyDB guide's §58](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) already modeled for its own limitations relative to real Tomcat.

---

# 53. Thread Pool Sizing for CPU-Bound Work

For work that spends nearly all its time actually computing (hashing, parsing, in-memory transformation), adding more threads than there are CPU cores to run them on adds pure context-switching overhead with no additional throughput — the [MiniTomcat guide's §18](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already derives this:

```text
corePoolSize ≈ number of CPU cores (Runtime.getRuntime().availableProcessors())
```

`maximumPoolSize` for genuinely CPU-bound work is often set equal to `corePoolSize` — there is nothing to gain from letting the pool grow further when the bottleneck is compute, not availability of threads to run on.

---

# 54. Thread Pool Sizing for I/O-Bound Work

For work that spends most of its time **waiting** — a database call, a downstream HTTP request — threads sit idle for the majority of their lifetime, so many more of them than there are cores can be usefully in flight at once:

```text
threads ≈ cores × (1 + waitTime / computeTime)
```

A pool handling requests that spend 90ms waiting on a database and 10ms doing real computation (`waitTime/computeTime = 9`) on an 8-core machine could reasonably support roughly `8 × (1 + 9) = 80` concurrent threads before CPU becomes the limiting factor — a dramatically different number from §53's CPU-bound sizing, for the same core count, purely because of what the threads spend their time doing.

---

# 55. Monitoring: Active Count, Queue Size, Completed Task Count

A pool that only exposes "is it working or not" is nearly impossible to tune correctly under real, changing load — a few cheap, always-on counters make the difference:

```java
public int getActiveCount() { return (int) workers.stream().filter(Worker::isRunningTask).count(); }
public int getQueueSize() { return workQueue.size(); }
public int getPoolSize() { return poolSize.get(); }
public long getCompletedTaskCount() { return completedTaskCount.get(); } // an AtomicLong, incremented in Worker.runTask (§19)
```

`getQueueSize()` trending upward over time, with `getPoolSize()` already at `maximumPoolSize`, is the single clearest real-time signal that a pool is genuinely under-provisioned for its current load — exactly the kind of metric §49's benchmarking-style reasoning (from the [ConcurrentHashMap guide](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>)) argues for measuring rather than guessing at.

---

# 56. Phase 24 — A Minimal ScheduledExecutorService

Some work isn't "run this as soon as possible" but "run this at a specific future time, or repeatedly on an interval" — a `ScheduledExecutorService`-shaped API needs a different kind of queue: one ordered by **when a task becomes eligible to run**, not by arrival order.

```java
public interface MiniScheduledExecutorService extends MiniExecutorService {
    MiniFuture<?> schedule(Runnable task, long delay, TimeUnit unit);
    MiniFuture<?> scheduleAtFixedRate(Runnable task, long initialDelay, long period, TimeUnit unit);
}
```

---

# 57. Implementing a DelayQueue From Scratch

```java
// scheduled/DelayQueue.java
public class DelayQueue<E extends Delayed> {
    private final PriorityQueue<E> heap = new PriorityQueue<>(); // ordered by getDelay() ascending — a min-heap by deadline
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();

    public void offer(E item) {
        lock.lock();
        try {
            heap.offer(item);
            if (heap.peek() == item) available.signalAll(); // the new item is now the SOONEST — wake anyone waiting
        } finally { lock.unlock(); }
    }

    public E take() throws InterruptedException {
        lock.lock();
        try {
            while (true) {
                E head = heap.peek();
                if (head == null) {
                    available.await(); // nothing scheduled at all — wait indefinitely for the first item
                } else {
                    long delayNanos = head.getDelay(TimeUnit.NANOSECONDS);
                    if (delayNanos <= 0) return heap.poll(); // it's due — take it now
                    available.awaitNanos(delayNanos);          // wait exactly until it's due, or until something newer arrives
                }
            }
        } finally { lock.unlock(); }
    }
}
```

A `PriorityQueue` ordered by deadline, plus `take()` computing **exactly** how long to wait for the soonest-due item (re-checking after every wake, per §13's "why `while` not `if`" discipline) is the entire mechanism — a `MiniScheduledExecutorService`'s workers (§16's `Worker`, unmodified) simply `take()` from this queue instead of §12's plain `MiniBlockingQueue`, and periodic tasks (`scheduleAtFixedRate`) re-`offer()` themselves with a new future deadline immediately after each run.

---

# 58. Work Stealing and ForkJoinPool: A Different Architecture Entirely

Everything built in this guide uses **one shared queue** every worker pulls from (§12–§18) — correct, but that single queue can itself become a contention point under very high submission rates, and it doesn't naturally handle a task that *spawns more tasks* (recursive divide-and-conquer work) efficiently. `ForkJoinPool` solves both with a genuinely different design: **each worker has its own local deque** of tasks, pushes and pops from its **own** end (`LIFO`, cheap, no contention with other workers), and only when a worker's own deque is empty does it **steal** a task from the **far end** of another worker's deque (`FIFO` from that worker's perspective, so the stealer takes the *oldest*, coarsest-grained task rather than colliding with the owner's own most-recent work). This is worth knowing exists — and *why* it exists (recursive task spawning wants each worker generating and consuming its own subtasks locally, only reaching across to another worker's queue when genuinely starved for work) — even though building it in full is a substantially larger undertaking than this guide's shared-queue design.

---

# 59. Virtual Threads as an Alternative to Pooling Platform Threads

The [MiniTomcat guide's §39–§40](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) already covers this in depth for the server-socket case, and the conclusion transfers directly here: `Executors.newVirtualThreadPerTaskExecutor()` sidesteps this entire guide's core/max/queue sizing model (§21–§26) by making thread creation so cheap that **pooling threads stops being necessary at all** — a new virtual thread per task, parked cheaply on blocking I/O rather than occupying a scarce platform thread. It doesn't make everything in this guide obsolete: `Future`/`FutureTask` (§30–§34), rejection-shaped backpressure concepts, and `@Async`-style declarative dispatch (§42–§52) are all still meaningful ideas regardless of what actually executes the work underneath — only the specific "how many platform threads, how do they grow and shrink" machinery in §21–§26 is what virtual threads let you stop hand-tuning.

---

# 60. Common Mistakes

- **Mistake 1 — An unbounded work queue.** Exactly [the MiniTomcat guide's §17](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) warning, repeated here because it's just as easy to get wrong in a hand-built pool: an unbounded queue means step 3 of §23's decision order never triggers, so `maximumPoolSize` is silently meaningless.
- **Mistake 2 — Swallowing exceptions inside a worker without a handler (§19).** Left unhandled, an exception in `task.run()` kills that worker thread outright, silently shrinking the pool one failure at a time.
- **Mistake 3 — Forgetting `while`, using `if`, around a `Condition.await()` (§13-§14, §57).** A single missed re-check after waking is enough to let a race corrupt the queue's invariants under real concurrent load.
- **Mistake 4 — Assuming `cancel(true)` always stops a running task (§33).** It only requests interruption — a task with no interruption checks and no interruptible blocking calls keeps running to completion regardless.
- **Mistake 5 — Calling `shutdown()` and immediately assuming the pool has stopped.** `shutdown()` is asynchronous with respect to in-flight work — `awaitTermination()` (§38) is what actually confirms termination.
- **Mistake 6 — Sharing one `@Async` executor for both fast, frequent work and slow, rare work (§50).** A single shared pool means the slow work can starve the fast work of threads — named, independently-sized pools per workload exist specifically to prevent this.
- **Mistake 7 — Expecting self-invoked `@Async` methods to run asynchronously (§52).** A call from inside the same class bypasses the proxy entirely and runs synchronously, silently.
- **Mistake 8 — Applying `@Async` to a class with no interface, expecting it to "just work" (§46, §52).** This guide's JDK-dynamic-proxy-based implementation cannot proxy a class with no interface at all — it will fail loudly (§48's explicit check) rather than silently, but only if that check is actually in place.

---

# 61. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| `MiniBlockingQueue` (§12-§15) | `put()` blocks when full, `take()` blocks when empty, both unblock correctly when the other happens | Multi-threaded test: fill the queue, start a blocked `put()` on another thread, `take()` one item, assert the blocked `put()` completes |
| Pool sizing decision order (§22-§26) | Tasks fill core threads first, then the queue, then grow to max, then reject — in exactly that order | Submit tasks one at a time with a controllable "how many are currently running" probe, asserting the pool's state after each submission matches §23's decision order |
| Rejection policies (§29) | Each policy does exactly what it claims once the pool and queue are both full | Saturate the pool deliberately (long-running tasks), submit one more, assert the policy-specific behavior |
| `FutureTask` state machine (§30-§34) | `cancel()` before running succeeds; `cancel(false)` on a running task fails; `get()` correctly wraps a thrown exception in `ExecutionException` | Unit tests targeting each state transition directly |
| Shutdown semantics (§35-§38) | `shutdown()` finishes queued work; `shutdownNow()` returns exactly the tasks that never ran | Queue several tasks, call each shutdown variant, assert on what actually executed vs. what was returned |
| `@Async` proxy behavior (§42-§52) | A void `@Async` method returns immediately and runs on the correct named pool; a `MiniFuture`-returning one is genuinely asynchronous; self-invocation is (correctly) synchronous | Integration test using real bean wiring, asserting on which thread actually executed the method body |

---

# 62. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Producer-Consumer** | Callers producing tasks, workers consuming them, `MiniBlockingQueue` as the buffer | §11-§18 |
| **Strategy** | `RejectedExecutionHandler`, and the choice of named executor per `@Async` method | §28-§29, §43, §50 |
| **State** | `MiniFutureTask`'s explicit `NEW/RUNNING/COMPLETED/FAILED/CANCELLED` state machine | §30-§34 |
| **Proxy** | The entire `@Async` mechanism — a generated stand-in object intercepting calls to a real one | §44-§48 |
| **Template Method** (implicit) | Every `Worker`'s loop follows the identical get-task/run-task/repeat shape regardless of what the task does | §16 |
| **Object Pool** (the executor itself, at the meta level) | Reusing a fixed set of worker threads instead of creating one per task | §16-§26 |

The **Proxy** pattern is worth calling out as this guide's most conceptually distinct addition — every other pattern here manages *threads and tasks*; Proxy is the one mechanism that makes `@Async` invisible to ordinary calling code, by inserting an entire extra layer of indirection at the object-reference level rather than at the concurrency level.

---

# 63. Progressive Interview Question Set

**Level 1 — Why a pool at all**
1. Why is `new Thread(task).start()` per submission dangerous at scale, concretely — not just "it's not scalable"?
2. What's the difference between `Runnable` and `Callable`, and why does `Future` need `Callable` specifically?

**Level 2 — The blocking queue**
3. Why does a correct blocking queue need `while`, not `if`, around its `Condition.await()` calls?
4. Why use two separate `Condition`s (`notEmpty`/`notFull`) instead of one shared wait-set?

**Level 3 — Pool sizing**
5. State, precisely, the order `execute()` makes its four-step decision, and explain why growing the pool is checked *after* the queue, not before.
6. Why don't core threads time out by default, while non-core ones do?

**Level 4 — Futures and cancellation**
7. Walk through `FutureTask`'s state machine for a task that's cancelled while already running.
8. Does `cancel(true)` guarantee a running task stops? Why or why not?

**Level 5 — Shutdown**
9. What's the concrete difference between what `shutdown()` and `shutdownNow()` do to a task currently sitting in the queue?

**Level 6 — @Async and dynamic proxies**
10. Explain, mechanically, what `Proxy.newProxyInstance` actually creates, and what role the `InvocationHandler` plays.
11. Why can't a JDK dynamic proxy proxy a class with no interface, and what technique does real Spring use instead?
12. Why does calling an `@Async` method from inside the same class (self-invocation) run synchronously instead of asynchronously?

**Final challenge:** Design a way to let a caller of an `@Async` method specify, per call (not per method declaration), which named executor should handle that specific invocation — without changing the method's own signature. What would you have to add to `AsyncInvocationHandler` (§47), and what new problem does making the executor choice dynamic-per-call introduce that a static `@Async("name")` annotation doesn't have?

---

# 64. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| CGLIB/ByteBuddy-based proxying | `@Async` support for classes with no interface | Replaces §44-§48's `java.lang.reflect.Proxy` call with a subclass-generation library, same `AsyncInvocationHandler` logic underneath |
| Work-stealing pool | Efficient handling of recursive, self-spawning tasks | The architecture sketched in §58, built out in full |
| Metrics/observability integration | Real-time pool health visible in a dashboard, not just via `getActiveCount()`-style polling | Wiring §55's counters into a metrics library, alongside per-task timing |
| Configurable rejection-then-retry | A hybrid rejection policy that retries with backoff before finally giving up | A `RejectedExecutionHandler` (§28) implementation combining §29's ideas with the [locking guide's retry-with-backoff pattern](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>) |
| Per-task priority | Some submitted tasks should run before others, not strictly FIFO | Replacing §12's plain circular-buffer queue with a `PriorityBlockingQueue`-style structure ordered by a task priority field |
| Virtual-thread-backed `@Async` executors | Combine §42-§52's declarative API with §59's scaling model | `AsyncExecutorRegistry` (§43) registering a virtual-thread-per-task executor under a name, with zero change to `AsyncInvocationHandler` itself |

---

# 65. Final Takeaway

Every mechanism in this guide answers the same underlying question at a different layer: **how do you let work happen without the caller waiting for it, while still bounding the resources that work consumes?** A blocking queue answers it between producers and workers (§11–§15); core/max sizing with idle timeout answers it for how many threads exist at all (§21–§26); a rejection policy answers it for the moment even that bound is exceeded (§27–§29); `Future`/`FutureTask` answers it for how a caller eventually collects a result without blocking until it's ready (§30–§34); and `@Async`'s dynamic proxy answers it for how "this call should be asynchronous" can be declared once, centrally, instead of scattered across every call site (§42–§52). None of these are independent tricks — they compose into exactly the shape `java.util.concurrent.ThreadPoolExecutor` and Spring's `@Async` already have, and having built each layer yourself is what turns "I can configure a thread pool" into "I know exactly what happens, mechanically, when I do."



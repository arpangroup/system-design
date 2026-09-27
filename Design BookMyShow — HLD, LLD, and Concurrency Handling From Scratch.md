# Design BookMyShow — HLD, LLD, and Concurrency Handling From Scratch

# 1. What We Are Building

```text
User selects a seat --> [ Seat Hold, TTL ] --pay--> [ Payment ] --confirm--> [ Booking ]
                              |                                                  |
                    (auto-released if unpaid)                          [ Seat: BOOKED ]
                              |
              Two users, SAME seat, SAME instant --> exactly ONE must win, ALWAYS
```

A movie/event ticket booking platform looks like a straightforward CRUD application — until two users click "book" on the same seat within milliseconds of each other, which is the one scenario an interviewer will always eventually steer toward, because it's where a design's actual correctness lives or dies. This guide builds one from scratch, with concurrency handling as the central subject rather than an afterthought: the exact anatomy of the double-booking race condition, pessimistic locking versus optimistic locking with retry, temporary seat holds with TTL expiry (and the subtle race between a hold expiring and a payment succeeding), idempotent payment confirmation, the flash-sale problem a single popular show creates at scale, a virtual waiting room for admission control, and real-time seat-map synchronization across every viewer's screen.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Explain the exact mechanics of a check-then-act race condition on a shared seat, and reproduce it as a concrete, minimal failing scenario.
- Implement and compare pessimistic locking (database row locks) and optimistic locking (version-based compare-and-swap) for seat booking, and articulate the real throughput-versus-simplicity tradeoff between them.
- Design temporary seat holds with TTL expiry, and correctly close the race between a hold expiring and a payment succeeding at nearly the same instant.
- Design idempotent payment confirmation that survives a retried or duplicated network callback without double-charging or double-booking.
- Diagnose why ordinary locking isn't sufficient for a flash-sale scenario (a blockbuster's tickets going on sale), and design a virtual waiting room that controls admission upstream of the booking system entirely.
- Design real-time seat-map synchronization so every viewer sees a seat someone else just booked change color within moments, without polling.
- Apply SOLID principles and recognizable design patterns (State, Strategy, Observer) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design BookMyShow" (or Ticketmaster, or Fandango — the same question wears many names) is a favorite precisely because its business logic is genuinely simple, which strips away any excuse for a candidate to hide behind domain complexity — the entire interview lives or dies on how correctly and how efficiently the design handles concurrent access to a scarce, shared resource: a specific seat, at a specific showtime. It rewards a candidate who can name the actual race condition precisely, reason correctly about the tradeoffs between locking strategies, and recognize that a well-known blockbuster's ticket release is a fundamentally different scale problem than ordinary steady-state traffic. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Records model immutable seat/booking snapshots precisely; sealed interfaces keep booking status exhaustive |
| Seat inventory store | Relational database with row-level locking support | `SELECT ... FOR UPDATE` and optimistic version columns are both first-class relational features |
| Temporary seat holds | Redis, with native key TTL | A hold that "expires on its own" is exactly what a TTL-backed key-value store is built for |
| Distributed locking (flash sales) | Redis-based distributed lock (Redlock-style) | Coordinates lock ownership across many application servers, not just threads on one machine |
| Admission control (flash sales) | Token-based virtual waiting room | Controls how many users even *reach* the booking system at once, independent of any locking strategy downstream |
| Real-time seat map sync | Pub/sub (Redis or a message broker) pushed to WebSocket clients | Every viewer's seat map reflects a booking within moments, without client-side polling |

---

# 5. Project Structure

```text
bookmyshow/
├── src/main/java/com/example/booking/
│   ├── domain/
│   │   └── Venue.java, Show.java, Seat.java, SeatState.java, Booking.java  // §10, §24
│   ├── concurrency/
│   │   ├── PessimisticSeatLock.java                                        // §17-18
│   │   ├── OptimisticSeatLock.java (version column)                        // §20-21
│   │   └── RetryPolicy.java (exponential backoff)                          // §23
│   ├── hold/
│   │   ├── SeatHoldService.java (Redis TTL)                                // §26-27
│   │   └── HoldExpiryReconciler.java                                       // §29
│   ├── payment/
│   │   ├── PaymentGateway.java                                            // §31
│   │   └── IdempotentConfirmationHandler.java                             // §32
│   ├── admission/
│   │   ├── VirtualWaitingRoom.java                                        // §37-38
│   │   └── DistributedSeatLock.java (Redlock-style)                       // §40
│   ├── pricing/
│   │   └── PricingStrategy.java (Strategy interface)                      // §46
│   └── realtime/
│       └── SeatMapPublisher.java (pub/sub)                                // §44
├── src/test/java/com/example/booking/
│   ├── DoubleBookingRaceTest.java
│   ├── OptimisticRetryTest.java
│   └── HoldExpiryPaymentRaceTest.java
└── artifact/
    └── concurrency-arena.html   -- the runnable simulator, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design a movie ticket booking platform like BookMyShow."*

Even for a platform everyone has personally used, the intentionally open prompt needs narrowing: is the interesting part the browsing/discovery experience, or specifically the booking and payment flow — because for this guide, and for most interviewers asking this question, it's the latter. How many seats does a typical show have, and how concentrated can demand get (a routine Tuesday matinee versus a midnight blockbuster premiere)? Is there a payment gateway involved, with its own network unreliability to design around? Does a user get any grace period to enter payment details after selecting a seat, or is it instantly booked? The answers reshape which parts of the design carry real weight, and this guide's own explicit focus — concurrency handling — is precisely the part worth spending the most interview time on.

---

# 7. Functional Requirements

- **Browse shows** by movie/event, venue, and showtime, and view a seat map showing which seats are currently available.
- **Select and hold a seat** temporarily while the user completes payment, without permanently reserving it if they abandon the flow.
- **Confirm a booking** only after successful payment, and release the hold if payment fails or times out.
- **Guarantee no seat is ever booked for two different users** — the single non-negotiable correctness requirement this entire guide is organized around.
- **Support high-demand releases** (a popular show's tickets going on sale at a known instant) without the booking system falling over.
- **Reflect bookings in real time** on every viewer's seat map, so a seat someone else just booked visibly changes state without a manual refresh.

---

# 8. Non-Functional Requirements

- **Correctness under concurrency is non-negotiable**: double-booking a seat is a business-critical failure, not a rare acceptable edge case — this requirement outranks throughput or latency whenever the two are in tension.
- **Low booking latency** under ordinary load: selecting and holding a seat should feel instantaneous to a single user with no contention.
- **Graceful degradation under extreme, concentrated demand**: a flash-sale spike must not corrupt data or crash the system, even if it means some users wait in a queue.
- **No double-charging**: a retried or duplicated payment confirmation must never result in a user being charged twice for the same booking.
- **Bounded hold duration**: a seat placed on hold must always become available again within a fixed, predictable time if the holding user never completes payment.
- **Real-time consistency**: the gap between a seat actually being booked and every viewer's UI reflecting that must be small enough that users rarely attempt to select an already-gone seat.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Venue** — a physical location containing one or more screens/halls, each with its own fixed seat layout.
- **Show** — a specific movie or event scheduled at a specific screen, at a specific date and time — the actual unit seats are booked *against* (the same physical seat is a completely independent bookable resource for every different show scheduled in that screen).
- **Seat** — a single bookable position within a show, with an explicit lifecycle (§24) rather than a bare boolean.
- **Seat Hold** — a temporary, time-bounded reservation of a seat, held by one user while they complete payment.
- **Booking** — the durable, confirmed record of a successfully paid seat, immutable once created.
- **Payment** — the external, unreliable dependency whose success or failure ultimately decides whether a hold becomes a booking or is released.

---

# 10. Identifying the Core Domain Entities

```java
public record Venue(String venueId, String name, List<Screen> screens) { }
public record Screen(String screenId, List<Seat> layout) { }
public record Show(String showId, String screenId, String movieTitle, Instant startsAt) { }

public enum SeatState { AVAILABLE, HELD, BOOKED }

public class Seat {
    private final String seatId;
    private final String showId; // a seat is scoped to ONE show, not the physical screen alone
    private final String label;  // e.g. "F12"
    private SeatState state;
    private long version;        // for optimistic locking, §20-21
}

public record Booking(String bookingId, String showId, List<String> seatIds, String userId, Instant confirmedAt) { }
```

Scoping `Seat` to a specific `showId` (rather than modeling one physical seat shared across every show ever scheduled in that screen) is the detail that makes the whole domain model correct — physical seat "F12" is a completely independent bookable resource for the 2pm showing and the 8pm showing of the same movie in the same room, and conflating them would make one show's booking incorrectly affect another's availability.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
User selects seat -->|   Seat Hold Service      |----(TTL, Redis, §26-27)
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |    Payment Gateway         |------>|  Idempotent Confirm.     |
                    +-----------+--------------+       |  Handler (§31-32)        |
                                |                        +------------------------+
                    +-----------v--------------+
                    |   Booking Confirmation      |
                    |  (Seat: HELD -> BOOKED)     |
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |    SeatMapPublisher         |------>|  Every connected client  |
                    |   (pub/sub, §44)            |       |  (real-time seat map)    |
                    +------------------------+       +------------------------+

                    +------------------------+
Flash-sale surge --->|  Virtual Waiting Room    |----(admission control, §36-38)
                    +------------------------+          BEFORE any of the above
```

Every mechanism this guide builds sits along one linear path — hold, pay, confirm, publish — with concurrency control (§16-23) and the flash-sale admission layer (§34-38) both acting as *gates* on this same path, never as separate systems bolted on afterward.

---

# 12. Follow-up Question 2 — "Why Does Booking a Seat Need Any Special Handling — Why Not Just Insert a Row and Check for Uniqueness?"

> **Interviewer:** *"A booking is just a row in a table. Add a uniqueness constraint on (show, seat). Doesn't that solve double-booking automatically?"*

A uniqueness constraint alone would correctly *reject* a second insert for an already-booked seat — but only if the check for availability and the act of booking happen as a single, indivisible operation. The actual bug lives one level earlier: a naive implementation typically **checks** whether a seat is available (a `SELECT`), and only *afterward* **acts** on that answer (an `INSERT` or `UPDATE`) — and between those two separate steps, another request can slip in and do the exact same thing, both having "seen" the seat as available before either one committed.

---

# 13. The Double-Booking Race Condition, Made Concrete

```text
Time    User A                              User B
t0      SELECT state FROM seats WHERE id=F12  -- sees AVAILABLE
t1                                            SELECT state FROM seats WHERE id=F12  -- ALSO sees AVAILABLE
t2      UPDATE seats SET state=BOOKED         
t3                                            UPDATE seats SET state=BOOKED
                                               -- BOTH requests believed they were the one booking a free seat
                                               -- BOTH succeed, unless something PREVENTS this specific interleaving
```

Both users genuinely saw the seat as available at the moment they checked — the bug isn't that either read was wrong, it's that nothing prevented a *second* read from happening before the *first* write had a chance to invalidate it. This exact shape — read, decide, write, with a window in between where another actor can interleave — is the single most common concurrency bug in booking-style systems, and every fix in this guide (§16-23, §26-29) is a different way of closing that window.

---

# 14. Follow-up Question 3 — "Walk Through the Naive 'Check-Then-Act' Bug Step by Step, in Code"

> **Interviewer:** *"Show me the actual code that has this bug, not just a diagram."*

```java
public class NaiveBookingService {
    public BookingResult bookSeat(String seatId, String userId) {
        Seat seat = seatRepository.findById(seatId);
        if (seat.state() != SeatState.AVAILABLE) {
            return BookingResult.rejected("Seat is no longer available");
        }
        // <-- ANOTHER THREAD/REQUEST CAN EXECUTE THIS ENTIRE METHOD, RIGHT HERE, BEFORE THE NEXT LINE RUNS
        seat.setState(SeatState.BOOKED);
        seatRepository.save(seat);
        return BookingResult.confirmed(seatId, userId);
    }
}
```

Nothing in this method is individually wrong — the check is correct, the write is correct — but the method as a whole is not **atomic**: two threads (or two application server instances) can both pass the `if` check before either one's `save` call takes effect, and both proceed to overwrite the same seat's state, each unaware the other is doing the identical thing at the identical moment.

---

# 15. Anatomy of a Check-Then-Act Race Condition

```text
This same shape appears constantly across distributed systems, always with the same fix
requirement: collapse "check" and "act" into ONE atomic operation, OR make the SECOND actor's
write fail/detect the conflict once it discovers the first actor already acted.

Two families of fix:
  PESSIMISTIC (§16-18):  prevent the second reader from even LOOKING until the first writer
                          is completely done -- serialize access up front.
  OPTIMISTIC (§19-23):   let both readers proceed freely, but make the SECOND writer's commit
                          FAIL once it discovers the underlying data already changed since it read.
```

Neither family is "the" correct answer in isolation — they represent a genuine tradeoff (§41-42) between guaranteeing no wasted work up front (pessimistic) versus maximizing throughput when contention is actually rare (optimistic), and a mature system frequently uses both, in different parts of the same flow.

---

# 16. Follow-up Question 4 — "Fix This with Database-Level Pessimistic Locking"

> **Interviewer:** *"Design the pessimistic-locking fix precisely — what does the database actually do differently?"*

By making the **read itself** acquire an exclusive lock on the row it reads, held until the current transaction commits or rolls back — any *other* transaction attempting to read that same row (specifically, to read it *for the purpose of writing*) is forced to **wait** until the first transaction finishes, rather than being allowed to proceed with a now-stale view of the data. This is exactly what `SELECT ... FOR UPDATE` does in a relational database: it's a `SELECT`, but one that says "and don't let anyone else touch this row until I'm done with it."

---

# 17. Pessimistic Locking: SELECT FOR UPDATE

```text
Time    User A (Transaction 1)                    User B (Transaction 2)
t0      BEGIN
t1      SELECT * FROM seats WHERE id=F12
        FOR UPDATE  -- acquires an EXCLUSIVE row lock
t2                                                 BEGIN
t3                                                 SELECT * FROM seats WHERE id=F12
                                                    FOR UPDATE  -- BLOCKS, waits for A's lock to release
t4      UPDATE seats SET state=BOOKED WHERE id=F12
t5      COMMIT  -- lock released HERE
t6                                                 -- B's SELECT FOR UPDATE now proceeds, but re-reads
                                                       the ROW AS IT NOW STANDS: state=BOOKED
t7                                                 -- B's own application code sees state=BOOKED and
                                                       correctly REJECTS the booking -- no race possible
```

The critical property is that B's `SELECT FOR UPDATE` doesn't just block — once it finally *does* proceed, it re-reads the row's **current** value, which by then correctly reflects A's completed write. This is what closes the race entirely: B never gets to act on a stale read, because it was never allowed to read at all until the row was safe to read.

---

# 18. Implementing Pessimistic Seat Locking

```java
public class PessimisticBookingService {
    @Transactional
    public BookingResult bookSeat(String seatId, String userId) {
        Seat seat = seatRepository.findByIdForUpdate(seatId); // SELECT ... FOR UPDATE, blocks concurrent readers
        if (seat.state() != SeatState.AVAILABLE) {
            return BookingResult.rejected("Seat is no longer available"); // now a TRUSTWORTHY read
        }
        seat.setState(SeatState.BOOKED);
        seatRepository.save(seat);
        return BookingResult.confirmed(seatId, userId);
        // transaction commits here -- ONLY NOW does the row lock release, and only after
        // the state change is already durably persisted
    }
}
```

Wrapping the entire method in a single `@Transactional` boundary is what makes the lock's lifetime match the operation's own lifetime exactly — the lock is held from the `FOR UPDATE` read all the way through the commit, guaranteeing no other transaction can observe or act on this seat in between.

---

# 19. Follow-up Question 5 — "Pessimistic Locking Serializes Everyone on a Busy Show. What's the Alternative?"

> **Interviewer:** *"On a popular show, hundreds of users are booking different seats simultaneously. If they're all forced to wait on row locks, doesn't that hurt throughput even when they're not actually conflicting with each other?"*

To be precise first: `SELECT ... FOR UPDATE` only locks the **specific rows** it reads, so two users booking two genuinely *different* seats never contend with each other at all — the throughput cost is real only when many users target the *same* seat, which is a narrow, specific kind of contention. But when that narrow case does happen frequently (a small number of highly desirable seats, all attempted simultaneously), an alternative avoids blocking entirely: **optimistic locking**, which lets every reader proceed immediately and instead detects conflicts at write time.

---

# 20. Optimistic Locking: Version Numbers and Compare-and-Swap

```text
Seat row: { id: F12, state: AVAILABLE, version: 7 }

User A: reads {state: AVAILABLE, version: 7}   -- no lock taken, read is instantaneous
User B: reads {state: AVAILABLE, version: 7}   -- ALSO no lock, both proceed freely

User A: UPDATE seats SET state=BOOKED, version=8 WHERE id=F12 AND version=7
        -- SUCCEEDS: the row's version WAS 7 when this ran, exactly as A expected

User B: UPDATE seats SET state=BOOKED, version=8 WHERE id=F12 AND version=7
        -- FAILS: 0 rows matched -- the row's version is now 8, NOT 7, because A already changed it
        -- B's application code checks the affected-row count, sees 0, and knows its
           read was STALE -- it must re-read and retry (§22-23), or simply report rejection
```

The `WHERE ... AND version=7` clause is the entire mechanism — it makes the write itself conditional on the row being *exactly* as the writer last observed it, which the database can check and enforce atomically as part of the single `UPDATE` statement, with zero explicit locking required.

---

# 21. Implementing Optimistic Locking with a Version Column

```java
public class OptimisticBookingService {
    public BookingResult bookSeat(String seatId, String userId) {
        Seat seat = seatRepository.findById(seatId); // plain read, NO lock
        if (seat.state() != SeatState.AVAILABLE) {
            return BookingResult.rejected("Seat is no longer available");
        }

        int rowsUpdated = seatRepository.updateStateIfVersionMatches(
            seatId, SeatState.BOOKED, seat.version(), seat.version() + 1);

        if (rowsUpdated == 0) {
            return BookingResult.conflict("Seat was modified concurrently"); // caller decides: retry or give up
        }
        return BookingResult.confirmed(seatId, userId);
    }
}
```

`updateStateIfVersionMatches` is the one method that must translate directly into a single conditional `UPDATE ... WHERE id = ? AND version = ?` statement — if this were instead implemented as a separate read-then-write in application code, it would silently reintroduce the exact same check-then-act race this whole strategy exists to eliminate.

---

# 22. Follow-up Question 6 — "Optimistic Locking Can Fail Under Contention and Needs a Retry. How Do You Handle That Cleanly?"

> **Interviewer:** *"A conflict isn't necessarily a real rejection — if the seat is still available, the losing request could just try again. How do you implement that retry correctly, and what if the retry also collides?"*

By retrying the **entire read-then-conditional-write sequence** from scratch — not just the write — since the whole point of the failure is that the previously-read version is now stale; retrying only the write with the same stale version would simply fail identically forever. Each retry re-reads the current state, checks availability fresh, and attempts the conditional write again — and because contention naturally *thins out* as some earlier competing requests succeed and stop retrying, a bounded number of retries with a brief backoff between attempts converges quickly in practice, even under real contention.

---

# 23. Retry Strategy and Exponential Backoff for Optimistic Conflicts

```java
public class RetryingOptimisticBookingService {
    private static final int MAX_ATTEMPTS = 5;

    public BookingResult bookSeatWithRetry(String seatId, String userId) {
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            BookingResult result = optimisticBookingService.bookSeat(seatId, userId);
            if (result.isConfirmed() || result.isRejected()) {
                return result; // a genuine rejection (seat truly gone) is NOT worth retrying
            }
            // result is a CONFLICT -- the seat MIGHT still be available, re-read and try again
            sleepWithJitter(baseDelayMs: 20, attempt);
        }
        return BookingResult.conflict("Too much contention -- please try again");
    }

    private void sleepWithJitter(long baseDelayMs, int attempt) {
        long backoff = baseDelayMs * (1L << attempt); // exponential
        long jitter = ThreadLocalRandom.current().nextLong(backoff / 2);
        try { Thread.sleep(backoff / 2 + jitter); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

Distinguishing a genuine **rejection** (the seat is definitively gone, confirmed by a fresh read) from a **conflict** (the write collided, but the seat's actual current availability is still unknown) is the detail that keeps this retry loop from either giving up too early on a seat that's actually still bookable, or endlessly retrying against a seat that's genuinely, permanently taken.

---

# 24. Class Diagram: The Seat/Booking Concurrency Core

```text
+------------------------+        +------------------------+
|          Seat             |------->|       SeatState          |
|  state, version           |       |  AVAILABLE|HELD|BOOKED   |
+-----------+--------------+        +------------------------+
            |
   +--------+--------+
   v                   v
+------------------------+        +------------------------+
| PessimisticBookingSvc    |        | OptimisticBookingSvc     |
| SELECT ... FOR UPDATE     |        | UPDATE ... WHERE version |
+------------------------+        +-----------+--------------+
                                                  |
                                    +-------------v-------------+
                                    | RetryingOptimisticBookingSvc|
                                    | (wraps, adds backoff, §23) |
                                    +------------------------+
```

Both concrete booking services expose the identical `bookSeat(seatId, userId) -> BookingResult` contract — a caller (the API layer, or the seat-hold flow in §26 onward) never needs to know or care which concurrency strategy is actually enforcing correctness underneath, which is exactly what lets a system mix both strategies for different parts of its traffic (§41-42) without duplicating the calling code.

---

# 25. Follow-up Question 7 — "A User Takes Five Minutes to Enter Payment Details. How Do You Avoid Locking the Seat Out for Everyone Else the Whole Time?"

> **Interviewer:** *"Neither pessimistic nor optimistic locking, as you've described them, holds a seat across the several minutes a user might spend on a payment form. What's actually needed here?"*

Neither locking strategy alone is the right tool for this — a database lock held for minutes would be a serious throughput problem, and optimistic locking alone offers no way to say "this seat is provisionally mine for the next few minutes" at all. What's needed is a distinct concept: a **seat hold** — a short-lived, explicitly time-bounded reservation that a user acquires the instant they select a seat, which automatically expires and releases the seat back to availability if they never complete payment within that window.

---

# 26. Seat Holds: Temporary Reservation with a TTL

```text
User selects seat F12  --> SeatHoldService.tryHold(seatId, userId, ttl=5 minutes)
                                |
                       SUCCESS: seat marked HELD, a Redis key "hold:F12" is set with a 5-MINUTE TTL
                                |
              +---------------+----------------+
              v                                  v
   User completes payment                User does NOTHING for 5 minutes
   within the window                              |
              |                          Redis key EXPIRES AUTOMATICALLY
   Booking confirmed,                              |
   seat -> BOOKED                          Seat reverts to AVAILABLE, with
                                            NO application code needing to
                                            run a cleanup job at all
```

Using a store with **native key expiry** (Redis's `EXPIRE`/`SETEX`) rather than a manually-scheduled cleanup job is the detail that makes this reliable — the release-on-timeout behavior is a guarantee the storage layer itself provides, not something that depends on a background job actually running on schedule, which is a meaningfully weaker guarantee if that job is ever delayed, crashed, or simply forgotten.

---

# 27. Implementing Seat Holds via Redis with Automatic Expiry

```java
public class SeatHoldService {
    private final RedisClient redis;
    private static final Duration HOLD_TTL = Duration.ofMinutes(5);

    public HoldResult tryHold(String seatId, String userId) {
        // SET key value NX EX <ttl> -- atomically: only succeeds if the key does NOT already exist
        boolean acquired = redis.setIfAbsent("hold:" + seatId, userId, HOLD_TTL);
        if (!acquired) {
            return HoldResult.rejected("Seat is already held or booked");
        }
        seatRepository.updateState(seatId, SeatState.HELD); // reflected in the durable store too, for the seat map
        return HoldResult.granted(seatId, userId, HOLD_TTL);
    }

    public void releaseHold(String seatId, String userId) {
        String holder = redis.get("hold:" + seatId);
        if (userId.equals(holder)) { // only the ACTUAL holder can release it -- §28 explains why this check matters
            redis.delete("hold:" + seatId);
            seatRepository.updateState(seatId, SeatState.AVAILABLE);
        }
    }
}
```

`SET ... NX` (set-if-not-exists) is doing the same atomic-conditional-write job here that a version-column `UPDATE` did for optimistic locking (§20-21) — two users attempting to hold the same seat at the same instant can never both succeed, because the underlying Redis command itself is atomic, with no separate check-then-set window for a race to hide in.

---

# 28. Follow-up Question 8 — "What If Payment Succeeds Right as the Hold Just Expired? How Do You Avoid Charging Someone for a Seat You Then Give Away?"

> **Interviewer:** *"A hold's TTL expires at the exact moment a payment gateway callback confirms success. The seat has already been released and possibly re-held by someone else. What happens to the first user's now-successful payment?"*

This is a genuinely subtle race, and the fix is to make payment confirmation **re-validate hold ownership at the moment of confirmation**, not merely trust that the hold was valid when payment *started* — if the hold has expired (and possibly been reassigned) by the time the payment gateway calls back, the confirmation must be rejected and the payment **refunded**, rather than blindly marking the seat booked for a user who no longer holds it, which could either double-book the seat or silently steal it from whoever holds it now.

---

# 29. Closing the Hold-Expiry-vs-Payment-Success Race

```java
public class HoldExpiryReconciler {
    public ConfirmationResult confirmBooking(String seatId, String userId, PaymentResult payment) {
        String currentHolder = redis.get("hold:" + seatId);

        if (!userId.equals(currentHolder)) {
            // the hold expired (or was reassigned) BEFORE this confirmation arrived --
            // the payment succeeded for a reservation that NO LONGER EXISTS
            paymentGateway.refund(payment.paymentId());
            return ConfirmationResult.failed("Your hold expired before payment completed -- refunded automatically");
        }

        redis.delete("hold:" + seatId);
        seatRepository.updateState(seatId, SeatState.BOOKED);
        bookingRepository.save(new Booking(generateId(), payment.showId(), List.of(seatId), userId, Instant.now()));
        return ConfirmationResult.confirmed(seatId);
    }
}
```

Automatically refunding, rather than simply rejecting and leaving the user to notice their payment went through for nothing, is what makes this failure mode acceptable rather than a genuine incident — a rare, honest "you were a moment too slow, here's your money back" is a wildly better outcome than either double-booking a seat or silently keeping a payment for a reservation that no longer exists.

---

# 30. Follow-up Question 9 — "A Payment Gateway Callback Can Be Retried or Duplicated by the Network. How Do You Avoid Double-Charging or Double-Confirming?"

> **Interviewer:** *"A payment gateway's confirmation webhook fires twice for the same payment — a common, expected occurrence over an unreliable network. What stops your system from booking two seats, or recording the payment twice?"*

By treating the payment gateway's own **payment ID** as an idempotency key: before acting on any confirmation callback, check whether a booking already exists for that exact payment ID — if it does, the callback is a duplicate, and the correct response is to return the *same* result as the first time, without re-executing any of the booking or state-changing logic a second time. This is precisely the same **Idempotency-Key** pattern this series' REST API guide detailed in its own §48-50, applied here to a payment webhook instead of a client-submitted request.

---

# 31. Idempotency for Payment Confirmation

```text
Payment gateway calls the confirmation webhook TWICE for payment "pay_9981" (network retry):

Call 1: bookingRepository.findByPaymentId("pay_9981") -> NOT FOUND
        -> proceed: confirm the hold, create the Booking, record payment_id="pay_9981"

Call 2 (duplicate): bookingRepository.findByPaymentId("pay_9981") -> FOUND (from Call 1)
        -> do NOT re-run any booking logic -- return the EXISTING booking's result, unchanged
```

The idempotency check must happen **before** any state-changing work, and the payment ID must be recorded as part of the **same** atomic write that creates the booking — if these were two separate steps, a crash between them would reopen exactly the race this mechanism exists to close.

---

# 32. Implementing Idempotent Booking Confirmation

```java
public class IdempotentConfirmationHandler {
    @Transactional
    public ConfirmationResult onPaymentWebhook(PaymentResult payment) {
        Optional<Booking> existing = bookingRepository.findByPaymentId(payment.paymentId());
        if (existing.isPresent()) {
            return ConfirmationResult.confirmed(existing.get()); // duplicate webhook -- return the ORIGINAL result
        }
        return holdExpiryReconciler.confirmBooking(payment.seatId(), payment.userId(), payment);
        // the actual INSERT inside confirmBooking includes payment_id as a UNIQUE column,
        // so even a race between two SIMULTANEOUS duplicate webhooks (not just sequential
        // retries) is closed by the database's own uniqueness constraint as a final backstop
    }
}
```

The unique constraint on `payment_id` at the database level is the detail that makes this correct even under genuine concurrency (two duplicate webhooks arriving at nearly the same instant, not just one arriving after the other) — the application-level `findByPaymentId` check handles the common sequential case cheaply, while the database constraint is what actually guarantees correctness in the rare, truly-concurrent case.

---

# 33. Class Diagram: The Seat Hold and Payment Flow

```text
+------------------------+        +------------------------+
|    SeatHoldService        |------->|      Redis (TTL)         |
|  tryHold(), releaseHold() |       |  SET NX EX, auto-expiry  |
+-----------+--------------+        +------------------------+
            |
            v
+------------------------+        +------------------------+
|   PaymentGateway          |------->|  IdempotentConfirmation  |
|  (external, unreliable)   |       |  Handler                 |
+------------------------+        +-----------+--------------+
                                                  |
                                                  v
                                    +------------------------+
                                    |  HoldExpiryReconciler     |
                                    |  confirmBooking()         |
                                    +-----------+--------------+
                                                  |
                                    +-------------v-------------+
                                    |        Booking              |
                                    |  (immutable once created)  |
                                    +------------------------+
```

`IdempotentConfirmationHandler` sits directly between the untrustworthy external payment gateway and the actual state-changing logic in `HoldExpiryReconciler` — every mechanism this guide has built so far (holds, TTL expiry, idempotency) composes along this exact path, each one responsible for exactly one failure mode and none of them duplicating another's job.

---

# 34. Follow-up Question 10 — "A Single Popular Show Sells Out in Seconds When Tickets Open. What Breaks at That Scale, Beyond Ordinary Locking?"

> **Interviewer:** *"A blockbuster's tickets go on sale at a known instant, and tens of thousands of users hit 'book' within the first few seconds. Your locking strategies are correct — so what actually breaks?"*

Correctness isn't the problem at this scale — pessimistic and optimistic locking both remain individually correct even under this load. The problem is **contention volume**: tens of thousands of requests competing for a few hundred seats means the overwhelming majority are guaranteed to fail, yet all of them still consume real capacity (database connections, application threads, network bandwidth) just to be told "sorry, gone" — the system can be *correct* and still fall over from sheer load, entirely apart from any locking bug.

---

# 35. The Flash-Sale Problem: When Locking Alone Isn't Enough

```text
Ordinary traffic:  requests roughly match available seats -- locking mostly just serializes
                    the rare genuine conflict, cheaply.

Flash-sale traffic: requests VASTLY exceed available seats (50,000 requests for 300 seats) --
                    even PERFECTLY correct locking means 49,700+ requests must each still
                    acquire a connection, attempt a lock, fail, and respond -- the sheer VOLUME
                    of doomed attempts is what saturates the database and application tier,
                    not any flaw in the locking logic itself.
```

The fix has to act **before** any of this contention ever reaches the booking system's locking layer at all — which is precisely the role of a virtual waiting room (§36-38): controlling *how many* requests are even allowed to attempt a booking at once, independent of whether the booking logic underneath is correct.

---

# 36. Follow-up Question 11 — "Design a Virtual Waiting Room That Controls Admission During a Flash Sale"

> **Interviewer:** *"Design the mechanism that decides which of those 50,000 users are even allowed to attempt a booking right now, and in what order."*

Every arriving user is issued a **queue token** immediately (cheap, no contention possible, since it touches nothing shared but an incrementing counter), and is only admitted into the actual booking flow once their token's position reaches the front of the queue at a rate the booking system can genuinely sustain — everyone else sees a simple "you're in line, position N" screen, never touching the seat map or locking layer until it's actually their turn.

---

# 37. The Virtual Waiting Room Pattern

```text
50,000 users arrive within the first second of tickets going on sale
        |
        v
[ Token issued instantly: "you are #14,382 in line" ] -- O(1), just an atomic counter increment
        |
        v
[ Admission controller lets N users through per second, N tuned to what the booking
  system can actually sustain without falling over ]
        |
        v
Only ADMITTED users ever reach the seat map / hold / booking flow at all --
everyone else is simply WAITING, consuming near-zero backend resources while they do
```

The queue token itself is nearly free to issue precisely because it requires no coordination beyond a single atomic increment — the expensive part of the whole system (locking, payment, database writes) is now only ever exercised by however many users the system can genuinely handle at once, regardless of how many are actually trying.

---

# 38. Implementing a Token-Based Admission Queue

```java
public class VirtualWaitingRoom {
    private final AtomicLong nextTokenNumber = new AtomicLong(0);
    private final AtomicLong nextAdmittedNumber = new AtomicLong(0);
    private final int admissionRatePerSecond;

    public QueueToken issueToken(String userId) {
        long position = nextTokenNumber.incrementAndGet(); // O(1), no lock contention beyond one atomic op
        return new QueueToken(userId, position, Instant.now());
    }

    public boolean isAdmitted(QueueToken token) {
        return token.position() <= nextAdmittedNumber.get();
    }

    // called on a fixed schedule, e.g. every second, by a single background process
    public void admitNextBatch() {
        nextAdmittedNumber.addAndGet(admissionRatePerSecond);
    }
}
```

`admissionRatePerSecond` is the one tunable knob translating directly into "how much load the booking system underneath is allowed to ever see at once" — set correctly (informed by real load testing of the locking/payment path), the booking system experiences smooth, sustainable traffic no matter how many total users are simultaneously waiting in line.

---

# 39. Follow-up Question 12 — "How Do You Distribute Lock Contention Across Many Servers, Not Just Many Threads on One Server?"

> **Interviewer:** *"Your booking service runs on many application server instances behind a load balancer. A database row lock coordinates threads within one process well enough — how does locking work when the contending requests land on entirely different machines?"*

A database-level lock (`SELECT ... FOR UPDATE`) already coordinates correctly across *any* number of application servers, since the lock genuinely lives in the shared database, not in any one server's memory — this is actually already solved by §16-18's design. Where a genuinely *distributed* lock becomes necessary is for coordination that has **no natural shared database row to anchor to** — for instance, coordinating the virtual waiting room's own admission-rate enforcement across multiple admission-controller instances, which is exactly the case Redis-based distributed locking (§40) addresses.

---

# 40. Distributed Locking with Redis (Redlock-Style)

```java
public class DistributedSeatLock {
    private final RedisClient redis;

    public boolean tryAcquire(String lockKey, String ownerId, Duration ttl) {
        // atomic: set the key ONLY if absent, with a TTL as a safety net in case the
        // lock holder crashes and never explicitly releases it
        return redis.setIfAbsent(lockKey, ownerId, ttl);
    }

    public void release(String lockKey, String ownerId) {
        // only release if WE are still the recorded owner -- prevents a slow process from
        // accidentally releasing a lock that has since expired and been reacquired by someone else
        String currentOwner = redis.get(lockKey);
        if (ownerId.equals(currentOwner)) redis.delete(lockKey);
    }
}
```

This is structurally identical to `SeatHoldService` (§27) — both are "atomic set-if-absent, with a TTL as a safety net, and an ownership check before release" — which is not a coincidence: a seat hold *is* a distributed lock, scoped specifically to one seat, with a business-meaningful TTL rather than a purely technical one.

---

# 41. Follow-up Question 13 — "How Do You Decide Between Pessimistic Locking, Optimistic Locking, and a Distributed Hold — What's the Actual Tradeoff?"

> **Interviewer:** *"You've now built three different coordination mechanisms. When would you actually reach for each one?"*

They solve genuinely different problems, not competing solutions to the same one: **pessimistic locking** suits a short, fast, single-database-transaction operation where contention is common enough that wasted optimistic retries would cost more than just waiting; **optimistic locking** suits the same short operation when contention is rare, letting the common uncontended case run at full speed with zero blocking; **a TTL-backed hold** suits an operation that spans real user think-time (minutes, not milliseconds) — a duration far too long for either database locking strategy to reasonably hold a lock across.

---

# 42. Comparing Concurrency Strategies: Correctness, Throughput, and Fairness

```text
                     Correctness    Throughput under      Throughput under      Fits long
                                    LOW contention          HIGH contention        user think-time
Pessimistic lock      Guaranteed     Good (lock is brief)    Poor (serializes)     No (lock held
                                                                                    far too long)
Optimistic lock       Guaranteed     Excellent (no lock      Poor (many wasted     No (same reason)
                                     taken at all)            retries)
TTL-backed hold       Guaranteed     Excellent                Excellent (admission  YES -- this is
  (+ waiting room)                                            queue absorbs the    exactly the
                                                                excess upstream)    problem it solves
```

All three guarantee correctness identically — the actual decision is entirely about which throughput/fairness profile matches the specific operation being protected, which is why this guide's booking flow (§26-29) uses a hold specifically for the user-facing "pick a seat, then pay" step, while a genuinely instantaneous internal state transition might reasonably use optimistic locking instead.

---

# 43. Follow-up Question 14 — "How Do You Keep the Seat Map in Sync — a User Should See a Seat Someone Else Just Booked Go Gray in Real Time"

> **Interviewer:** *"Two users are looking at the same seat map. One books F12. How does the OTHER user's screen find out, without polling the server every second?"*

By having the booking system **publish** an event the instant a seat's state actually changes, to every client currently subscribed to that show's seat map — a **publish/subscribe** channel, with each connected client holding an open connection (a WebSocket) the server can push to directly, rather than every client repeatedly asking "has anything changed yet?" on a fixed interval.

---

# 44. Real-Time Seat Map Updates via Pub/Sub

```java
public class SeatMapPublisher {
    private final RedisPubSub redisPubSub;

    public void onSeatStateChanged(String showId, String seatId, SeatState newState) {
        redisPubSub.publish("seatmap:" + showId, new SeatUpdateEvent(seatId, newState, Instant.now()));
    }
}

// each application server subscribes to shows its connected clients are actively viewing,
// and relays incoming events to the SPECIFIC WebSocket connections watching that show
public class SeatMapWebSocketRelay {
    public void onRedisEvent(String showId, SeatUpdateEvent event) {
        connectedClientsFor(showId).forEach(ws -> ws.send(event));
    }
}
```

Publishing through Redis pub/sub (rather than each application server only notifying its own directly-connected clients) is what makes this work correctly across a fleet of many application servers — a booking processed by *any* server instance is broadcast to *every* server, each of which then relays it only to the specific clients it happens to be holding a WebSocket connection for.

---

# 45. Follow-up Question 15 — "How Do You Support Pricing Tiers and Discounts?"

> **Interviewer:** *"Different seat categories (premium, standard) cost different amounts, and promotional discounts apply sometimes. How does that fit into the design without complicating the booking/concurrency logic you've built?"*

By keeping pricing entirely **orthogonal** to concurrency control — a `PricingStrategy` computes a price given a seat and a show, and is consulted only when displaying a price or finalizing a payment amount, never touching seat state or locking at all. This is the same **Strategy** pattern this series has used repeatedly (Tic-Tac-Toe's AI difficulty, the Parking Lot guide's pricing schemes) — swapping in a promotional discount strategy requires zero changes to any of this guide's concurrency mechanisms.

---

# 46. Pluggable Pricing Strategy

```java
public interface PricingStrategy {
    BigDecimal priceFor(Seat seat, Show show);
}

public class TieredPricingStrategy implements PricingStrategy {
    private final Map<String, BigDecimal> priceByCategory; // e.g. "PREMIUM" -> 450, "STANDARD" -> 250

    @Override
    public BigDecimal priceFor(Seat seat, Show show) {
        return priceByCategory.getOrDefault(seat.category(), priceByCategory.get("STANDARD"));
    }
}

public class PromotionalDiscountStrategy implements PricingStrategy {
    private final PricingStrategy base; // DECORATES another strategy, exactly like the Parking Lot guide's own §24
    private final BigDecimal discountFraction;

    @Override
    public BigDecimal priceFor(Seat seat, Show show) {
        return base.priceFor(seat, show).multiply(BigDecimal.ONE.subtract(discountFraction));
    }
}
```

`PromotionalDiscountStrategy` wraps another `PricingStrategy` rather than reimplementing tiered pricing itself — the same Decorator-flavored composition this series has used consistently, letting a discount be layered on top of *any* base pricing scheme without duplicating it.

---

# 47. Class Diagram: Full Booking System

```text
+------------------------+       +------------------------+       +------------------------+
|   VirtualWaitingRoom      |------>|    SeatHoldService        |------>|  PaymentGateway          |
|  (admission control)      |       |  (TTL-backed, §26-27)     |       +-----------+--------------+
+------------------------+       +------------------------+                       |
                                                                                    v
+------------------------+       +------------------------+       +------------------------+
|    PricingStrategy        |       |   SeatMapPublisher         |<------|  IdempotentConfirmation  |
|  (orthogonal, §45-46)     |       |   (pub/sub, §44)           |       |  Handler                 |
+------------------------+       +------------------------+       +-----------+--------------+
                                                                                    v
                                                                        +------------------------+
                                                                        |        Booking             |
                                                                        +------------------------+
```

Tracing this diagram left to right is the entire booking flow: admission control gates entry, a hold reserves the seat, payment happens, confirmation is idempotent, a booking is durably created, and the seat map is published to every viewer — with pricing sitting entirely off to the side, consulted but never coordinating with any of it.

---

# 48. Capacity Estimation: Peak Concurrent Booking Attempts

```text
Assume: a blockbuster release, 500 screens nationwide, each averaging 150 seats, tickets for
        the first week open simultaneously at a single announced instant

Total bookable seats released at once = 500 * 150                    = 75,000 seats
Assume peak concurrent ARRIVAL rate in the first 10 seconds = 200,000 users attempting to book

Without a waiting room: 200,000 requests hit the booking/locking layer nearly simultaneously --
  even at a generous 2,000 sustained booking-attempts/sec the database can actually handle,
  clearing this backlog takes 100 SECONDS of pure queueing delay, during which EVERY new
  arrival adds to an already-saturated system -- a classic cascading-overload spiral.

With a waiting room, admission tuned to the database's real sustainable rate (2,000/sec):
  the SAME 200,000 users all get an instant token and an honest queue position, but the
  booking/locking layer itself only ever sees requests it can actually handle -- no overload,
  no cascading failure, just an honest (bounded) wait for whoever is far back in line.
```

This is the concrete, numeric justification for §36-38's entire design: the waiting room doesn't make the underlying seats available any faster, it makes the difference between "the system degrades gracefully into a visible queue" and "the system falls over entirely under load it was never going to be able to satisfy anyway."

---

# 49. Full Worked Example: Two Users, One Seat, Traced End to End

```text
1. Alice and Bob are both viewing the same show's seat map; both click seat F12 within
   200ms of each other (a genuine, realistic race)
2. Alice's request reaches SeatHoldService.tryHold("F12", "alice") FIRST by a few milliseconds:
     a. redis.setIfAbsent("hold:F12", "alice", 5min) -- SUCCEEDS (§27)
     b. Seat F12's durable state updated to HELD; SeatMapPublisher broadcasts this immediately (§44)
3. Bob's request reaches SeatHoldService.tryHold("F12", "bob") a moment later:
     a. redis.setIfAbsent("hold:F12", "bob", 5min) -- FAILS, key already exists (§27)
     b. Bob's client immediately sees "Seat no longer available" -- AND, independently, Bob's
        seat map already started turning gray from step 2b's broadcast, likely before he even
        finished clicking (§44)
4. Alice enters payment details over the next 90 seconds; her hold has 5 minutes, comfortably enough
5. Payment gateway calls back with a successful charge for Alice:
     a. IdempotentConfirmationHandler checks bookingRepository.findByPaymentId(...) -- not found,
        proceeds (§31-32)
     b. HoldExpiryReconciler confirms Alice is STILL the current holder of "hold:F12" -- yes (§29)
     c. Seat F12 -> BOOKED, Booking record created, hold key deleted
6. SeatMapPublisher broadcasts F12's final BOOKED state -- Bob's screen (and everyone else's)
   updates once more, now showing it as permanently taken, not merely temporarily held
```

Every mechanism this guide introduced via a follow-up question appears somewhere in this one race's resolution — the atomic hold, real-time broadcast, idempotent confirmation, and ownership re-validation are not independent, optional features, they are the actual steps that correctly resolved one genuine, realistic two-user collision.

---

# 50. Final Architecture Diagram

```text
Flash-sale traffic --> [ VirtualWaitingRoom ] --admitted-->
                                                                 +------------------------+
Ordinary traffic ------------------------------------------->  |   Seat Map / Booking API |
                                                                 +-----------+--------------+
                                                                             |
                                                               +-------------v-------------+
                                                               |     SeatHoldService          |
                                                               |    (Redis, TTL, §26-27)      |
                                                               +-------------+-------------+
                                                                             |
                                                               +-------------v-------------+
                                                               |      PaymentGateway           |
                                                               +-------------+-------------+
                                                                             |
                                                               +-------------v-------------+
                                                               | IdempotentConfirmationHandler|
                                                               |  -> HoldExpiryReconciler     |
                                                               +-------------+-------------+
                                                                             |
                                                    +------------------------+------------------------+
                                                    v                                                   v
                                        +------------------------+                       +------------------------+
                                        |        Booking DB         |                       |    SeatMapPublisher      |
                                        |  (payment_id UNIQUE, §32) |                       |    (pub/sub -> all       |
                                        +------------------------+                       |     connected clients)  |
                                                                                            +------------------------+
```

---

# 51. Design Patterns Used Throughout This Guide

- **Strategy** — `PricingStrategy` (§45-46) and the interchangeable concurrency approaches (§16-23: pessimistic vs. optimistic) both let a policy vary independently of the code that invokes it.
- **Decorator** — `PromotionalDiscountStrategy` (§46) wraps another `PricingStrategy` to layer a discount on top of any base scheme without duplicating it; `RetryingOptimisticBookingService` (§23) does the same for retry behavior over the base optimistic service.
- **State** — `SeatState` (§10, §24) encodes exactly which transitions a seat can legally undergo (available → held → booked), rather than a bare boolean that any code could set inconsistently.
- **Facade** — `HoldExpiryReconciler.confirmBooking` (§29) hides ownership re-validation, state transition, and booking creation behind a single entry point the payment-confirmation path calls.
- **Observer** — `SeatMapPublisher` (§44) broadcasts state changes to every subscribed client without the booking logic needing to know how many viewers exist or what they do with the update.

---

# 52. SOLID Principles Applied

- **Single Responsibility** — `SeatHoldService` only manages holds; `IdempotentConfirmationHandler` only guards against duplicate confirmations; `PricingStrategy` only computes a price — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new concurrency strategy, pricing scheme, or admission policy each means adding a new implementation, never modifying the booking flow's own orchestration.
- **Liskov Substitution** — every `PricingStrategy` implementation must honestly return a price for any valid seat/show pair, so a discount decorator can wrap any base strategy interchangeably.
- **Interface Segregation** — `PricingStrategy` exposes exactly one method, so a trivial tiered-pricing implementation isn't forced to depend on discount-specific concepts only a decorator actually needs.
- **Dependency Inversion** — the booking API depends on `SeatHoldService`, `PaymentGateway`, and `PricingStrategy` as abstractions, never on concrete Redis or gateway implementations directly, so any of them can be swapped without touching the flow that calls them.

---

# 53. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Checking availability, then writing, as two separate steps  Atomic check-and-write, either locking family (§12-23)
Holding a database row lock across user think-time            TTL-backed hold in Redis instead (§25-27)
Confirming a booking without re-checking hold ownership       Re-validate at confirmation time (§28-29)
Trusting a payment webhook fires exactly once                 Idempotency keyed on payment ID (§30-32)
Relying on locking ALONE to survive a flash-sale spike         Admission control upstream of locking (§34-38)
Polling the server for seat-map updates                        Push via pub/sub to connected clients (§43-44)
Coupling pricing logic into the booking/concurrency path        Pricing kept fully orthogonal (§45-46)
```

---

# 54. Testing Strategy

- **Double-booking race tests** — fire many concurrent booking attempts at the identical seat (simulated or via real concurrent threads) and assert exactly one succeeds, regardless of which concurrency strategy is under test.
- **Optimistic retry tests** — deliberately induce a version conflict and assert the retry loop correctly re-reads fresh state rather than retrying with the same stale version.
- **Hold-expiry race tests** — simulate a hold expiring at nearly the same instant a payment confirmation arrives, and assert the confirmation is correctly rejected (and refunded) rather than silently booking a seat the user no longer holds.
- **Idempotency tests** — deliver the same payment webhook twice and assert the second delivery produces the identical result without executing any state-changing logic again.
- **Admission-control tests** — simulate arrival volume far exceeding the configured admission rate and assert the booking layer itself never receives more concurrent load than its configured sustainable rate, regardless of total demand.

---

# 55. Suggested Future Enhancements

- **Partial group bookings with atomicity** — letting a user book multiple seats in one transaction, requiring either *all* selected seats to succeed or the whole request to roll back, rather than partial success leaving a user with a scattered, undesired seat arrangement.
- **Dynamic admission-rate tuning** — automatically adjusting the virtual waiting room's admission rate based on real-time observed booking-layer latency, rather than a single statically-configured rate.
- **Seat recommendation** — suggesting a comparable alternative seat automatically when a user's first choice is lost to a race, reducing the friction of the rare-but-real hold-conflict case.
- **Cross-region booking consistency** — for a platform operating across multiple geographic regions, ensuring a seat's hold state is consistent even if a user's requests are routed to different regional deployments across the hold-then-pay window.
- **Fraud and bot-detection integration** — layering rate-limiting and behavioral analysis in front of the waiting room itself, since a flash-sale's admission queue is also exactly the resource a scalper's bot most wants to exploit.

---

# 56. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Describe the exact double-booking race condition in a naive "check then write" implementation, with a concrete two-user timeline. (§12-15)
2. Design and implement pessimistic locking for seat booking, and explain precisely why the transaction boundary must span both the read and the write. (§16-18)
3. Design optimistic locking as an alternative, including a correct retry strategy for a real conflict. (§19-23)
4. A user takes several minutes to complete payment. Design a mechanism that reserves their seat for that window without holding a database lock the whole time. (§25-27)
5. Design what happens when a hold expires at nearly the same instant a payment succeeds — don't just say "reject it," explain exactly what must happen to the payment itself. (§28-29)
6. A payment gateway webhook fires twice for the same payment. Design the fix, and identify the two independent layers of protection involved. (§30-32)
7. A blockbuster's tickets sell out in seconds. Explain why correct locking alone isn't sufficient at that scale, and design the fix. (§34-38)
8. Design how every viewer's seat map stays current in real time without client-side polling. (§43-44)

---

# 57. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **a "read, then act" sequence is only as correct as whatever prevents another actor from acting in between** — pessimistic locking prevents the second read from happening at all (§16-18); optimistic locking lets the read happen freely but makes the second write fail once it discovers the read was stale (§19-23); a TTL-backed hold extends this same guarantee across a window of real user time no database lock could reasonably span (§25-29); idempotency keys close the same gap for a payment gateway's own unreliable retries (§30-32); and a virtual waiting room recognizes that at sufficient scale, even a perfectly correct "read, then act" sequence needs to be rationed, not merely protected. Recognizing exactly where an interleaving window exists — and choosing the narrowest, cheapest mechanism that actually closes it — is the transferable skill this guide is really teaching.

---

# Design a Parking Lot System — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Vehicle arrives --> [ Entry Gate ] --issues--> [ Ticket ]
                          |
                   [ Spot Allocator ] --assigns--> [ Parking Spot ]
                          |
                   (later) Vehicle returns to [ Exit Gate ]
                          |
                   [ Pricing Strategy ] --computes--> Fee
                          |
                   [ Payment Processor ] --charges--> Receipt, spot freed
```

A parking lot system manages a physical lot's finite set of parking spots across one or more levels, issuing a ticket when a vehicle enters, assigning it an appropriate spot based on vehicle type and current availability, computing a fee based on duration and a configurable pricing scheme when the vehicle exits, and freeing the spot for the next vehicle. This guide builds one from scratch: clean domain modeling of vehicles/spots/tickets, efficient spot allocation without a linear scan, pluggable pricing strategies, a correct state machine for spot/ticket lifecycles, thread-safe concurrent gate operations, real-time availability reporting, and scaling considerations for a multi-level, multi-location deployment.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Model a parking lot's core domain entities (vehicle, spot, ticket, gate) so that adding a new vehicle type or spot type requires no changes to existing, already-tested code.
- Design an efficient spot allocation mechanism that avoids scanning every spot on every request, and reason about the best-fit vs. nearest-available tradeoff.
- Design a pluggable pricing strategy (hourly, flat-rate, vehicle-type-differentiated) using the Strategy pattern.
- Model a parking spot's and a ticket's lifecycle as an explicit state machine, preventing invalid states (e.g., double-assigning an occupied spot) by construction rather than by scattered `if` checks.
- Design thread-safe spot reservation so multiple concurrent entry gates never assign the same physical spot to two different vehicles.
- Design real-time availability reporting (a display board showing free spots per level) using the Observer pattern.
- Apply SOLID principles and recognizable design patterns (Strategy, State, Factory, Observer, Singleton) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design a parking lot" is a staple object-oriented design interview question precisely because it's small enough to fully implement in an interview's time budget, yet rich enough to expose whether a candidate defaults to a tangle of `if/else` chains on type checks, or reaches naturally for the small set of design patterns (Strategy, State, Factory, Observer) that keep a system like this extensible. It also has a genuine concurrency dimension (two gates assigning spots simultaneously) and a genuine scaling dimension (one lot vs. a multi-location chain) that a thorough interview will probe. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Sealed interfaces and records model vehicle/spot hierarchies precisely; pattern matching keeps type-specific logic exhaustive and compiler-checked |
| Pricing | Strategy pattern (in-process) | New pricing schemes plug in without touching gate logic |
| Concurrency | `java.util.concurrent` locks per spot/level | Fine-grained locking avoids serializing unrelated spots' assignment behind one global lock |
| Persistence (large-scale) | Relational store for tickets/transactions, in-memory cache for live availability counts | Tickets/payments need durability and consistency; availability counts are read far more often than written and tolerate brief staleness |
| Multi-location coordination | Independent per-lot instances, centrally aggregated for reporting | Each lot's gate operations must stay low-latency and local; only aggregate reporting needs cross-lot coordination |

---

# 5. Project Structure

```text
parking-lot/
├── src/main/java/com/example/parking/
│   ├── domain/
│   │   └── Vehicle.java, ParkingSpot.java, Ticket.java, SpotType.java  // §10, §13
│   ├── allocation/
│   │   ├── SpotAllocator.java (Strategy)                              // §16
│   │   └── BestFitAllocationStrategy.java                              // §18
│   ├── gate/
│   │   ├── EntryGate.java                                              // §21
│   │   └── ExitGate.java                                               // §27
│   ├── pricing/
│   │   ├── PricingStrategy.java (Strategy)                             // §23
│   │   └── HourlyPricingStrategy.java, FlatRatePricingStrategy.java     // §24
│   ├── state/
│   │   └── SpotState.java (State pattern)                              // §30
│   ├── concurrency/
│   │   └── SpotReservationManager.java                                 // §34
│   └── notification/
│       └── AvailabilityNotifier.java (Observer)                        // §37
└── src/test/java/com/example/parking/
    ├── SpotAllocationTest.java
    ├── ConcurrentEntryGateTest.java
    └── PricingStrategyTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design a parking lot system."*

Before drawing any boxes, the intentionally vague prompt needs narrowing: How many levels, and how many spot types (compact, large, motorcycle, handicapped, electric-vehicle-charging)? Is pricing flat, hourly, or does it vary by vehicle type or time of day? Is this a single physical lot, or must the design support a chain of many lots across a city, reported on centrally? Does payment happen at the exit gate, or does the system need to support pre-payment/reservations? The answers reshape which parts of the design need the most rigor, and asking them signals the difference between reciting a memorized class diagram and actually designing *this* system.

---

# 7. Functional Requirements

- **Issue a ticket** when a vehicle enters, recording its entry time and assigned spot.
- **Allocate an appropriate spot** based on vehicle type (a motorcycle shouldn't occupy a large spot unnecessarily; a large vehicle cannot fit in a compact spot at all).
- **Track real-time availability** of spots, per type and per level.
- **Compute a fee** at exit, based on parked duration and a configurable pricing scheme.
- **Process payment** and free the spot once the vehicle exits.
- **Support multiple simultaneous entry and exit gates**, operating concurrently without conflict.
- **Display available spot counts** (e.g., on an entrance display board) that stay current as vehicles enter and exit.

---

# 8. Non-Functional Requirements

- **Low latency at the gate**: a driver should not wait more than a second or two for a ticket to be issued or a spot to be assigned, since a queue of physical cars is accumulating in real time.
- **Correctness under concurrency**: two gates operating at the same instant must never assign the same physical spot to two different vehicles.
- **Consistency of availability counts**: the displayed count of free spots must never significantly diverge from the true count for long, even under high entry/exit throughput.
- **Extensibility**: adding a new vehicle type, spot type, or pricing scheme should require adding new code, not modifying existing, already-tested classes.
- **Durability of tickets and payment records**: a ticket and its associated payment record must survive a system restart — losing this data is a business-critical failure, not a cosmetic one.
- **Horizontal scalability (multi-location)**: the design must extend from one lot to many independently-operating lots without requiring a single, shared, tightly-coupled coordination point for routine entry/exit operations.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Vehicle** — the physical car/motorcycle/truck entering the lot, with a type that constrains which spots it can occupy.
- **Parking Spot** — a single physical space, with a type (compact/large/motorcycle/handicapped) and a current state (free/occupied/reserved/out-of-service).
- **Ticket** — the record created at entry: which vehicle, which spot, entry timestamp, and (once computed) exit timestamp and fee.
- **Gate** — an entry or exit point; an entry gate issues tickets and triggers allocation, an exit gate triggers fee computation and payment.
- **Pricing Strategy** — the configurable rule that turns a ticket's parked duration (and vehicle type) into a fee.
- **Level** — a physical floor of the lot, itself containing a collection of spots, used for both allocation locality and availability reporting granularity.

---

# 10. Identifying the Core Domain Entities

```java
public enum VehicleType { MOTORCYCLE, COMPACT, LARGE }
public enum SpotType { MOTORCYCLE, COMPACT, LARGE, HANDICAPPED }

public record Vehicle(String licensePlate, VehicleType type) { }

public class ParkingSpot {
    private final String spotId;
    private final SpotType type;
    private final int level;
    private SpotState state; // State pattern, §29-30

    public boolean canFit(Vehicle vehicle) {
        return switch (vehicle.type()) {
            case MOTORCYCLE -> true; // a motorcycle fits in ANY spot type, though it SHOULDN'T waste a large one (§18)
            case COMPACT -> type == SpotType.COMPACT || type == SpotType.LARGE;
            case LARGE -> type == SpotType.LARGE;
        };
    }
}

public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot assignedSpot;
    private final Instant entryTime;
    private Instant exitTime;   // null until the vehicle exits
    private BigDecimal fee;     // null until computed at exit
}
```

`canFit` deliberately lives on `ParkingSpot`, not as a separate lookup table or a chain of `if` statements inside the allocator — this keeps the compatibility rule co-located with the type it's actually a property of, and is the first small design decision that pays off directly in §13's Open/Closed discussion.

---

# 11. High-Level Architecture Overview

```text
                    +------------------+
Vehicle arrives --->|   Entry Gate      |
                    +--------+---------+
                             |
                    +--------v---------+       +------------------------+
                    |  SpotAllocator     |------>|   Level -> free-spot   |
                    |  (Strategy)        |       |   pools, per SpotType  |
                    +--------+---------+       +------------------------+
                             |
                    +--------v---------+
                    |     Ticket         |  (created, persisted)
                    +--------+---------+
                             |
                     ... vehicle parks, time passes ...
                             |
                    +--------v---------+       +------------------------+
Vehicle returns --->|   Exit Gate        |------>|   PricingStrategy       |
                    +--------+---------+       +------------------------+
                             |
                    +--------v---------+
                    | Payment Processor  | --> Receipt, spot freed, AvailabilityNotifier fires (§36-37)
                    +------------------+
```

Every gate operation flows through exactly two pluggable, swappable decision points — `SpotAllocator` (§14-18) at entry, `PricingStrategy` (§22-24) at exit — which is deliberate: these are precisely the two points most likely to need customization (different lots may allocate differently, or charge differently) without needing to touch the entry/exit gate orchestration logic itself.

---

# 12. Follow-up Question 2 — "How Do You Model Different Vehicle Types and Spot Types Cleanly?"

> **Interviewer:** *"A naive implementation checks `if (vehicle.type == MOTORCYCLE) { ... } else if (vehicle.type == COMPACT) { ... }` scattered across several classes. What's wrong with that, and what's the fix?"*

Scattering type checks across multiple classes means adding a new vehicle type (e.g., an oversized truck) requires hunting down and modifying every single one of those scattered conditionals — a direct **Open/Closed Principle** violation, since existing, already-tested code must change to accommodate a new type rather than simply being extended. The fix demonstrated in §10 — an enum plus a single method (`canFit`) that encapsulates the compatibility rule in one place — means a new vehicle or spot type only requires updating that one `switch`, with the compiler's exhaustiveness checking (in a sealed/enum `switch`) actively catching any type combination left unhandled.

---

# 13. Modeling Vehicle Types and Spot Types (Enum vs. Class Hierarchy Tradeoff)

```text
Enum-based (this guide's choice):
  VehicleType { MOTORCYCLE, COMPACT, LARGE }  -- compatibility logic lives in ONE method (canFit)
  Good when: the SET of types is stable, and behavior differences are simple compatibility rules.

Class-hierarchy-based (an alternative):
  abstract class Vehicle { abstract boolean fitsIn(SpotType type); }
  class Motorcycle extends Vehicle { ... }  class Truck extends Vehicle { ... }
  Good when: EACH vehicle type has substantially different, non-trivial BEHAVIOR beyond
  a simple compatibility check (e.g., different loading/unloading procedures, different
  fee multipliers baked into the type itself) -- otherwise it's unnecessary ceremony.
```

This guide deliberately chooses the enum-based approach because vehicle-type-specific *behavior* in a parking lot is genuinely thin (mostly just spot compatibility and a pricing multiplier, §24) — reaching for a full class hierarchy here would be exactly the kind of premature abstraction this series' own house style avoids; the class-hierarchy alternative becomes the *better* choice only once a vehicle type's behavior meaningfully diverges beyond simple data-driven rules.

---

# 14. Follow-up Question 3 — "How Does the System Decide Which Spot to Assign, Efficiently, Without Scanning Every Spot?"

> **Interviewer:** *"A naive allocator loops over every spot in the lot looking for a free one matching the vehicle's type. What's wrong with that at scale, and what's the fix?"*

A linear scan over every spot costs `O(total spots)` per allocation, even though the overwhelming majority of those spots are irrelevant to the search (wrong type, or already occupied) — at a large multi-level lot with thousands of spots, this becomes a real, unnecessary latency cost paid on every single entry, precisely when a physical queue of cars is waiting. The fix is to maintain **pre-partitioned pools of currently-free spots, indexed by type** (and optionally by level), so an allocation request only ever searches within the specific, already-narrow pool relevant to that vehicle.

---

# 15. Spot Allocation: Avoiding a Linear Scan

```text
Naive:  for each spot in ALL spots: if spot.isFree() && spot.canFit(vehicle): assign it
        -- O(total spots) per allocation, most of which are irrelevant to this specific request

This guide's approach:  maintain a Map<SpotType, Deque<ParkingSpot>> of CURRENTLY FREE spots,
        pre-partitioned by type. An allocation for a COMPACT vehicle only ever searches the
        (much smaller) compact-spots-currently-free pool -- O(1) removal from a deque, not
        a scan over the whole lot.
```

Maintaining these pools incrementally (removing a spot when it's assigned, adding it back when it's freed at exit) is what keeps allocation itself cheap — the cost of "finding a free spot of the right type" is paid once, continuously, as spots change state, rather than being re-derived by scanning everything on every single request.

---

# 16. Implementing a Spot Allocation Strategy

```java
public interface SpotAllocator {
    Optional<ParkingSpot> allocate(Vehicle vehicle);
    void release(ParkingSpot spot);
}

public class PooledSpotAllocator implements SpotAllocator {
    private final Map<SpotType, Deque<ParkingSpot>> freeSpotsByType;

    @Override
    public Optional<ParkingSpot> allocate(Vehicle vehicle) {
        for (SpotType candidateType : compatibleTypesFor(vehicle.type())) {
            Deque<ParkingSpot> pool = freeSpotsByType.get(candidateType);
            if (pool != null && !pool.isEmpty()) {
                ParkingSpot spot = pool.pollFirst(); // O(1) removal -- this spot is now considered assigned
                return Optional.of(spot);
            }
        }
        return Optional.empty(); // lot (or this vehicle's compatible spot types) is genuinely full
    }

    @Override
    public void release(ParkingSpot spot) {
        freeSpotsByType.get(spot.type()).addLast(spot); // returned to its pool, now allocatable again
    }

    private List<SpotType> compatibleTypesFor(VehicleType vehicleType) {
        // ORDER matters here -- see §17-18's best-fit discussion
        return switch (vehicleType) {
            case MOTORCYCLE -> List.of(SpotType.MOTORCYCLE, SpotType.COMPACT, SpotType.LARGE);
            case COMPACT -> List.of(SpotType.COMPACT, SpotType.LARGE);
            case LARGE -> List.of(SpotType.LARGE);
        };
    }
}
```

`SpotAllocator` is declared as an interface specifically so the allocation *policy* (which pool to try first, in what order) is swappable independently of the gate logic that calls it — the next follow-up question is precisely about what policy `compatibleTypesFor`'s ordering should encode.

---

# 17. Follow-up Question 4 — "A Vehicle Could Fit in a Smaller Spot, but Only Larger Ones Are Free — Best-Fit vs. Nearest-Available?"

> **Interviewer:** *"A motorcycle can fit in a motorcycle spot, a compact spot, or a large spot. If only large spots are free, do you assign one? And if so, does that waste capacity for a large vehicle that arrives next?"*

This is a genuine tradeoff, not a single correct answer: a **best-fit** policy always prefers the tightest-fitting available spot type first (motorcycle spot, then compact, then large, in that order) — preserving larger spots for vehicles that actually need them, at the cost of occasionally rejecting a motorcycle when only large spots remain free even though one technically fits. A **nearest-available** policy instead prioritizes minimizing walking distance or gate-to-spot travel time, potentially assigning a large spot to a motorcycle if it's simply closer, accepting the capacity inefficiency in exchange for driver convenience.

---

# 18. Best-Fit vs. Nearest-Available Allocation Strategy

```java
public class BestFitAllocationStrategy implements SpotAllocator {
    // compatibleTypesFor() ordered SMALLEST-fitting-type-first, exactly as shown in §16 --
    // this IS the best-fit policy, achieved purely through pool search ORDER, no extra logic needed
}

public class NearestAvailableAllocationStrategy implements SpotAllocator {
    private final Map<SpotType, Deque<ParkingSpot>> freeSpotsByType;
    private final ProximityIndex proximityIndex; // spots ordered by distance from the entry gate

    @Override
    public Optional<ParkingSpot> allocate(Vehicle vehicle) {
        return compatibleTypesFor(vehicle.type()).stream()
            .flatMap(type -> freeSpotsByType.getOrDefault(type, new ArrayDeque<>()).stream())
            .min(Comparator.comparingDouble(proximityIndex::distanceFromEntry)); // closest wins, REGARDLESS of type-fit tightness
    }
}
```

Both strategies implement the identical `SpotAllocator` interface introduced in §16 — a textbook **Strategy** pattern application — which means a lot operator can switch between best-fit and nearest-available (or even mix them, e.g., best-fit during low-occupancy periods and nearest-available once the lot nears capacity) via configuration alone, with zero changes to `EntryGate`'s own logic.

---

# 19. Follow-up Question 5 — "What Happens at the Entry Gate, Exactly, Step by Step?"

> **Interviewer:** *"Walk me through the exact sequence of operations from the moment a vehicle pulls up to the entry gate."*

Four ordered steps, each of which can independently fail in a way the design must account for: (1) capture the vehicle's identity (license plate, type — via a sensor, camera, or manual entry), (2) ask the `SpotAllocator` for a compatible free spot, failing gracefully (a "lot full" signal) if none exists, (3) create and persist a `Ticket` linking the vehicle, the assigned spot, and the entry timestamp, and (4) mark the assigned spot's state as occupied and physically open the gate. The ordering here matters: the spot must be marked occupied (§29-30) *before* the gate physically opens, so no other vehicle or process can be assigned that same spot in the brief window between allocation and physical occupancy.

---

# 20. Entry Flow: Ticket Issuance

```text
1. Vehicle arrives, identity captured (license plate, type)
2. EntryGate calls SpotAllocator.allocate(vehicle)
     -> spot found: proceed to step 3
     -> spot NOT found (lot full for this vehicle's compatible types): reject, display "FULL", do not proceed
3. Ticket created: { vehicle, assignedSpot, entryTime = now() }, persisted durably
4. assignedSpot.state transitions FREE -> OCCUPIED (§29-30)
5. Physical gate opens, vehicle proceeds to its assigned spot
```

Persisting the ticket **before** transitioning the spot's state (not after) is a deliberate ordering choice — if the system crashes between steps 3 and 4, recovery can safely re-derive "this spot must be occupied" from the persisted ticket's existence, whereas the reverse ordering (state change first, ticket persisted second) would leave a spot marked occupied with no corresponding ticket if a crash occurred in between.

---

# 21. Implementing the Entry Gate / Ticket Issuance

```java
public class EntryGate {
    private final SpotAllocator spotAllocator;
    private final TicketRepository ticketRepository;

    public Optional<Ticket> processEntry(Vehicle vehicle) {
        Optional<ParkingSpot> assignedSpot = spotAllocator.allocate(vehicle);
        if (assignedSpot.isEmpty()) {
            return Optional.empty(); // lot full for this vehicle type -- gate stays closed, no ticket issued
        }

        Ticket ticket = new Ticket(UUID.randomUUID().toString(), vehicle, assignedSpot.get(), Instant.now());
        ticketRepository.save(ticket);          // persisted FIRST (§20's ordering rationale)
        assignedSpot.get().markOccupied();      // THEN the spot's state transitions (§29-30)
        return Optional.of(ticket);
    }
}
```

`EntryGate` depends only on the `SpotAllocator` interface and a `TicketRepository` abstraction, never on a concrete allocation strategy or storage technology — precisely the **Dependency Inversion** this guide's SOLID discussion (§49) revisits, applied here to keep gate orchestration logic stable while its dependencies vary.

---

# 22. Follow-up Question 6 — "How Do You Compute the Fee — Flat Rate, Per-Hour, or Something More Nuanced?"

> **Interviewer:** *"Different lots price differently — some charge a flat daily rate, some charge per hour with a grace period, some charge more for larger vehicles. How do you support all of these without a tangle of conditionals in the exit gate?"*

By extracting fee computation into its own pluggable **Strategy**, exactly the same architectural move already used for spot allocation (§16-18) — `ExitGate` calls a `PricingStrategy.computeFee(ticket)` method without knowing or caring whether the concrete implementation is hourly, flat-rate, or vehicle-type-differentiated, which means adding a new pricing scheme (e.g., a weekday/weekend differential) never requires modifying `ExitGate` itself.

---

# 23. Designing a Pluggable Pricing Strategy

```java
public interface PricingStrategy {
    BigDecimal computeFee(Ticket ticket, Instant exitTime);
}
```

This single-method interface is deliberately narrow — **Interface Segregation** in action — so that a simple flat-rate implementation isn't forced to depend on, or even be aware of, concepts like grace periods or vehicle-type multipliers that only a more elaborate implementation actually needs.

---

# 24. Implementing Pricing Strategies

```java
public class HourlyPricingStrategy implements PricingStrategy {
    private final BigDecimal ratePerHour;
    private final Duration gracePeriod; // e.g., first 15 minutes free

    @Override
    public BigDecimal computeFee(Ticket ticket, Instant exitTime) {
        Duration parked = Duration.between(ticket.entryTime(), exitTime);
        if (parked.compareTo(gracePeriod) <= 0) {
            return BigDecimal.ZERO;
        }
        long billableHours = parked.minus(gracePeriod).toHours() + 1; // ROUND UP to the next full hour
        return ratePerHour.multiply(BigDecimal.valueOf(billableHours));
    }
}

public class VehicleTypeDifferentiatedPricingStrategy implements PricingStrategy {
    private final PricingStrategy baseStrategy;              // DELEGATES to another strategy, e.g. HourlyPricingStrategy
    private final Map<VehicleType, BigDecimal> typeMultipliers;

    @Override
    public BigDecimal computeFee(Ticket ticket, Instant exitTime) {
        BigDecimal baseFee = baseStrategy.computeFee(ticket, exitTime);
        BigDecimal multiplier = typeMultipliers.getOrDefault(ticket.vehicle().type(), BigDecimal.ONE);
        return baseFee.multiply(multiplier);
    }
}
```

`VehicleTypeDifferentiatedPricingStrategy` **wraps** another `PricingStrategy` rather than reimplementing hourly-rate logic itself — a small **Decorator**-flavored composition that lets vehicle-type differentiation be layered on top of *any* base pricing scheme (hourly or flat-rate) without duplicating that base scheme's own logic.

---

# 25. Follow-up Question 7 — "What Happens at Exit, Step by Step, Including Payment?"

> **Interviewer:** *"Walk me through the exact sequence from the moment a vehicle pulls up to the exit gate."*

Four ordered steps, mirroring entry's structure: (1) look up the ticket (via a scanned ticket ID, or license-plate lookup for lost-ticket handling, §38-39), (2) compute the fee via the configured `PricingStrategy`, using the current time as the exit timestamp, (3) process payment, and only upon successful payment (4) free the assigned spot (transitioning its state back to available) and physically open the gate. Payment succeeding **before** the spot is freed is the critical ordering detail — freeing the spot first would let another vehicle be assigned it while the exiting vehicle is still physically parked there, pending payment resolution.

---

# 26. Exit Flow: Fee Calculation and Payment

```text
1. Vehicle arrives at exit, ticket identified (scanned ID or license-plate lookup)
2. ExitGate calls PricingStrategy.computeFee(ticket, now())  -> fee amount
3. Payment processed for the computed fee
     -> SUCCESS: proceed to step 4
     -> FAILURE: gate stays closed, vehicle remains "parked" from the system's point of view
4. Ticket updated: exitTime = now(), fee = computed amount, persisted
5. assignedSpot.state transitions OCCUPIED -> FREE, spot RETURNED to SpotAllocator's free pool (§16)
6. Physical gate opens, vehicle departs
```

Step 5's spot release is what makes the spot available for the *next* vehicle's allocation — this is the moment `SpotAllocator.release(spot)` (§16) is actually called, closing the loop between a spot being taken out of its free pool at entry and returned to it at exit.

---

# 27. Implementing the Exit Gate / Payment Processing

```java
public class ExitGate {
    private final PricingStrategy pricingStrategy;
    private final PaymentProcessor paymentProcessor;
    private final SpotAllocator spotAllocator;
    private final TicketRepository ticketRepository;

    public PaymentResult processExit(String ticketId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        Instant exitTime = Instant.now();
        BigDecimal fee = pricingStrategy.computeFee(ticket, exitTime);
        PaymentResult result = paymentProcessor.charge(ticket.vehicle(), fee);

        if (!result.successful()) {
            return result; // gate stays closed -- spot is NOT freed until payment succeeds
        }

        ticket.recordExit(exitTime, fee);
        ticketRepository.save(ticket);
        ticket.assignedSpot().markFree();
        spotAllocator.release(ticket.assignedSpot()); // now allocatable to the NEXT vehicle
        return result;
    }
}
```

`ExitGate`'s dependency list mirrors `EntryGate`'s structure exactly (an allocator, a repository, plus a pricing strategy and payment processor) — both gates are thin orchestrators over pluggable collaborators, never owning business logic (allocation policy, pricing rules) directly themselves.

---

# 28. Follow-up Question 8 — "How Do You Model the Lifecycle of a Spot and a Ticket Cleanly, Avoiding Invalid States?"

> **Interviewer:** *"What stops your code from accidentally marking an already-occupied spot as occupied again, or freeing a spot that was never actually assigned?"*

By making a spot's state an explicit, small, closed set of values with clearly defined legal transitions between them — rather than a loose boolean flag (`isOccupied`) that any code, anywhere, could flip in any order. A `SpotState` modeled via the **State** pattern encodes *which transitions are even legal* directly into the state objects themselves, so calling `markOccupied()` on an already-occupied spot is a **compile-time-visible, structurally-encoded illegal operation**, not merely a runtime bug waiting to be triggered by a missed `if` check somewhere.

---

# 29. Modeling Spot and Ticket Lifecycle as a State Machine

```text
ParkingSpot states:
  FREE ----assign()----> OCCUPIED ----release()----> FREE
  FREE ----reserve()---> RESERVED ----assign()------> OCCUPIED
  FREE/OCCUPIED ----takeOutOfService()----> OUT_OF_SERVICE ----bringBackInService()----> FREE

Illegal transitions (structurally PREVENTED, not just discouraged by convention):
  OCCUPIED.assign()          -- already occupied, cannot be assigned again
  FREE.release()             -- was never occupied, nothing to release
  OUT_OF_SERVICE.assign()    -- must be brought back in service first
```

Each state above is a genuinely different set of *legal next operations* — `OCCUPIED` doesn't even expose an `assign()` method that does anything but reject the call (or, in a stricter design, doesn't expose it at all) — which is precisely what distinguishes this from a boolean flag: a boolean has exactly one bit of information and no way to express "this operation isn't valid from this state" beyond an ad hoc `if` check scattered wherever the flag happens to be read.

---

# 30. Implementing the State Pattern for ParkingSpot

```java
public interface SpotState {
    SpotState assign();          // returns the NEW state, or throws if illegal from this state
    SpotState release();
    SpotState takeOutOfService();
}

public class FreeState implements SpotState {
    @Override public SpotState assign() { return new OccupiedState(); }
    @Override public SpotState release() { throw new IllegalStateException("Spot is already free"); }
    @Override public SpotState takeOutOfService() { return new OutOfServiceState(); }
}

public class OccupiedState implements SpotState {
    @Override public SpotState assign() { throw new IllegalStateException("Spot is already occupied"); }
    @Override public SpotState release() { return new FreeState(); }
    @Override public SpotState takeOutOfService() { throw new IllegalStateException("Cannot take an occupied spot out of service directly"); }
}

public class ParkingSpot {
    private SpotState state = new FreeState();

    public void markOccupied() { this.state = state.assign(); }   // delegates the TRANSITION to the state itself
    public void markFree() { this.state = state.release(); }
}
```

Each concrete `SpotState` implementation owns exactly the transitions legal from that specific state — `ParkingSpot` itself never contains a single `if (currentState == ...)` check; it simply delegates every operation to whichever state object it currently holds, and that object either performs the transition or throws, entirely encapsulating the state machine's rules inside the states themselves rather than in `ParkingSpot`'s own methods.

---

# 31. Class Diagram: The Core Parking Lot Domain Model

```text
+------------------------+        +------------------------+
|      Vehicle             |        |     ParkingSpot          |
|  licensePlate, type      |<-------|  spotId, type, level     |
+------------------------+        |  state: SpotState        |
                                    +-----------+--------------+
                                                |
                                                v
                                    +------------------------+
                                    |      SpotState           |
                                    |    <<interface>>          |
                                    |  assign(), release()      |
                                    +-----------+--------------+
                                         ^      ^      ^
                                    +--------+ +--------+ +----------------+
                                    | Free   | | Occupied| | OutOfService   |
                                    | State  | | State   | | State          |
                                    +--------+ +--------+ +----------------+

+------------------------+        +------------------------+
|       Ticket             |------->|      SpotAllocator       |
|  vehicle, assignedSpot,  |       |    <<interface>>          |
|  entryTime, exitTime, fee|       +------------------------+
+------------------------+
            ^
            |
+------------------------+        +------------------------+
|      EntryGate           |       |      ExitGate            |
|  processEntry(vehicle)   |       |  processExit(ticketId)   |
+------------------------+        +-----------+--------------+
                                                |
                                                v
                                    +------------------------+
                                    |    PricingStrategy       |
                                    |    <<interface>>          |
                                    +------------------------+
```

`Ticket` is the central object binding a `Vehicle` to a `ParkingSpot` for the duration of one visit — both `EntryGate` and `ExitGate` operate on it at their respective ends of its lifecycle, and it's the natural unit of persistence (§20's ordering discussion is specifically about `Ticket` being durably saved before any state transition occurs).

---

# 32. Follow-up Question 9 — "How Do You Support Multiple Concurrent Entry/Exit Gates Without Double-Assigning the Same Spot?"

> **Interviewer:** *"A real lot has several entry gates operating simultaneously. Two vehicles arrive at two different gates at the exact same instant — what stops both from being assigned the same last remaining compact spot?"*

The `Deque.pollFirst()` call in §16's `PooledSpotAllocator` is not, by itself, safe under concurrent access from multiple gate threads unless the underlying `Deque` implementation itself guarantees atomicity for that operation — and even if the removal itself is atomic, the larger "check available, then act" sequence elsewhere in the system needs the same discipline. The fix is to use a genuinely thread-safe concurrent collection for each pool, and to ensure the "remove from pool" step is the single, atomic point of truth for whether a spot was actually won by a given request.

---

# 33. Concurrency: Avoiding Double-Assignment of the Same Spot

```text
Race WITHOUT proper concurrency control:
  Gate A: reads freeSpotsByType[COMPACT] -- sees spot C7 available
  Gate B: reads freeSpotsByType[COMPACT] -- ALSO sees spot C7 available (read happened before A's removal)
  Gate A: assigns C7 to vehicle 1
  Gate B: assigns C7 to vehicle 2  <-- DOUBLE-ASSIGNED, a genuine correctness bug

Fix: the POOL ITSELF must offer an atomic "take one item out, or tell me there are none" operation --
  a ConcurrentLinkedDeque's pollFirst() IS atomic at the single-call level, so as long as
  allocation NEVER does a separate "check isEmpty()" followed by a separate "pollFirst()"
  call (which would reintroduce the exact race above between the two calls), this is safe.
```

The lesson generalizes beyond this specific data structure: **the atomic unit of work must match the actual invariant being protected** — a check-then-act sequence split across two separate calls to a thread-safe collection is *not* itself thread-safe, even though each individual call is; §16's `PooledSpotAllocator.allocate()` was written from the start as a single `pollFirst()` call precisely to avoid ever needing a separate check.

---

# 34. Implementing Thread-Safe Spot Reservation

```java
public class PooledSpotAllocator implements SpotAllocator {
    // ConcurrentLinkedDeque: pollFirst() is atomic; no external synchronization needed for THIS operation
    private final Map<SpotType, ConcurrentLinkedDeque<ParkingSpot>> freeSpotsByType;

    @Override
    public Optional<ParkingSpot> allocate(Vehicle vehicle) {
        for (SpotType candidateType : compatibleTypesFor(vehicle.type())) {
            ConcurrentLinkedDeque<ParkingSpot> pool = freeSpotsByType.get(candidateType);
            if (pool == null) continue;
            ParkingSpot spot = pool.pollFirst(); // ATOMIC: either this call wins the spot, or the pool was already empty
            if (spot != null) {
                return Optional.of(spot);
            }
        }
        return Optional.empty();
    }
}
```

Notice there is no explicit `synchronized` block anywhere in this implementation — correctness here comes entirely from choosing a data structure (`ConcurrentLinkedDeque`) whose single operations already provide the exact atomicity guarantee needed, rather than from manually coordinating locks around a non-atomic collection, which would be both slower (broader lock scope) and easier to get subtly wrong.

---

# 35. Follow-up Question 10 — "How Do You Show Real-Time Available Spot Counts to Drivers Approaching the Lot?"

> **Interviewer:** *"A display board at the lot's entrance shows 'Compact: 12 available, Large: 3 available.' How does that board stay current as vehicles enter and exit throughout the day?"*

Rather than having the display board *poll* the allocator repeatedly asking "what's changed," the allocator should **push** an update to any interested listener every time a spot's availability actually changes (assigned or released) — this is the **Observer** pattern, and it means adding a second display board, or a mobile app showing the same live counts, requires only registering a new listener, never modifying `SpotAllocator`'s own allocation logic.

---

# 36. Real-Time Availability Display via Observer

```text
SpotAllocator.allocate() succeeds  -->  notify all registered AvailabilityListeners: "COMPACT count decreased by 1"
SpotAllocator.release() called     -->  notify all registered AvailabilityListeners: "COMPACT count increased by 1"

Listeners (each independent, none aware of the others):
  - EntranceDisplayBoard   (updates a physical LED sign)
  - MobileAppPushService   (sends a live update to a companion app)
  - OccupancyMetricsExporter (feeds a dashboard/monitoring system, analogous to this series'
                               own Real-Time Analytics Platform and Logging/Monitoring guides)
```

Every listener reacts to the exact same underlying event (an availability count changing for a given type) without the allocator needing to know how many listeners exist or what any of them actually do with the notification — precisely the decoupling payoff the Observer pattern has delivered consistently throughout this guide series (the Logging guide's `AlertingEngine`, the Analytics Platform's alerting engine).

---

# 37. Implementing the Availability Notifier

```java
public interface AvailabilityListener {
    void onAvailabilityChanged(SpotType type, int newFreeCount);
}

public class ObservableSpotAllocator implements SpotAllocator {
    private final SpotAllocator delegate; // wraps §34's PooledSpotAllocator -- a Decorator, not a reimplementation
    private final List<AvailabilityListener> listeners;
    private final Map<SpotType, AtomicInteger> freeCounts;

    @Override
    public Optional<ParkingSpot> allocate(Vehicle vehicle) {
        Optional<ParkingSpot> result = delegate.allocate(vehicle);
        result.ifPresent(spot -> {
            int newCount = freeCounts.get(spot.type()).decrementAndGet();
            listeners.forEach(l -> l.onAvailabilityChanged(spot.type(), newCount));
        });
        return result;
    }

    @Override
    public void release(ParkingSpot spot) {
        delegate.release(spot);
        int newCount = freeCounts.get(spot.type()).incrementAndGet();
        listeners.forEach(l -> l.onAvailabilityChanged(spot.type(), newCount));
    }
}
```

`ObservableSpotAllocator` **wraps** the underlying allocator rather than modifying it directly — another Decorator, exactly the same compositional technique §24 used for vehicle-type pricing — which means availability notification can be added to (or removed from) the system without touching `PooledSpotAllocator`'s own already-tested allocation logic at all.

---

# 38. Follow-up Question 11 — "How Do You Handle a Lost Ticket?"

> **Interviewer:** *"A driver arrives at the exit gate without their ticket. What does the system do?"*

Since the ticket's own ID is unavailable, the system falls back to an alternate lookup key — typically the vehicle's license plate, captured at entry (§10's `Vehicle` record already carries it) — to find the corresponding `Ticket` record, and then applies a configurable **lost-ticket policy**: commonly, a flat penalty fee added on top of the normally-computed fee (compensating for the operational cost of manual verification), rather than either refusing the vehicle exit entirely or waiving the fee outright.

---

# 39. Handling Lost Tickets and Edge Cases

```java
public class ExitGate {
    // ... existing fields ...
    private final BigDecimal lostTicketPenalty;

    public PaymentResult processExitByLicensePlate(String licensePlate) {
        Ticket ticket = ticketRepository.findActiveByLicensePlate(licensePlate)
            .orElseThrow(() -> new NoActiveTicketException(licensePlate));

        Instant exitTime = Instant.now();
        BigDecimal fee = pricingStrategy.computeFee(ticket, exitTime).add(lostTicketPenalty);
        // ... remainder identical to processExit(), §27 ...
        return paymentProcessor.charge(ticket.vehicle(), fee);
    }
}
```

Other edge cases worth naming explicitly (even if not fully implemented here): a vehicle that never actually left after its ticket's fee was paid (requiring a grace period before the spot is forcibly repossessed), and a spot mistakenly marked out-of-service while a vehicle is still parked in it (requiring the state machine from §29-30 to explicitly forbid `takeOutOfService()` from the `OccupiedState`, exactly as already shown).

---

# 40. Follow-up Question 12 — "How Do You Extend This to a Multi-Level, Multi-Location Deployment?"

> **Interviewer:** *"This all works for one lot with one level. How does the design change for a multi-level garage, or a chain of lots across a city, all reported on centrally?"*

Multi-level is a small, incremental extension: `ParkingSpot` already carries a `level` field (§10), and the free-spot pools (§15-16) can simply be further partitioned by level, or a level can be treated as its own independent sub-allocator whose results are aggregated. Multi-*location* is architecturally more significant: each physical lot should run as an **independent, self-contained instance** of everything built so far (its own gates, its own allocator, its own local availability state) — routine entry/exit operations at one location must never depend on network connectivity to a central system, since a lot's core function (letting a car in or out) cannot be allowed to fail just because a central reporting service is briefly unreachable.

---

# 41. Scaling to Multi-Level and Multi-Location Deployments

```text
Single lot (this guide's core design):  gates, allocator, and availability state ALL LOCAL to one lot.

Multi-level:  ParkingSpot.level partitions the free-spot pools further; allocation can
              prefer levels closer to the entrance (an extension of §18's nearest-available idea).

Multi-location (a chain of lots):
  Lot A (independent instance) --\
  Lot B (independent instance) ---+--> periodically PUSHES its own availability summary -->
  Lot C (independent instance) --/                                                          \
                                                                                       [ Central Aggregation Service ]
                                                                                                |
                                                                                 City-wide "find a nearby open lot" API
```

Each lot instance owning its own gates and allocator, and only *asynchronously pushing* summary availability data outward (rather than routing every entry/exit decision through a central system), is precisely the same architectural principle the Real-Time Analytics Platform guide's multi-tenancy design (§44-45 there) and the Rate Limiter guide's per-server fallback (§39 there) both rely on: a system's most latency-critical, most failure-sensitive operations should depend on the least possible cross-network coordination.

---

# 42. Follow-up Question 13 — "How Would This Look in a Real Large-Scale Deployment — Database, Caching, and So On?"

> **Interviewer:** *"Sketch the actual persistence and caching layers for a single, busy lot handling thousands of vehicles a day."*

Tickets and payment records need genuine **durability and transactional consistency** (a lost or double-charged payment is a real business and legal problem) — a relational database is the natural fit, with `Ticket` and its payment record written in the same transaction. Live availability *counts*, by contrast, are read far more often than they're written (every driver approaching the lot's display board reads them; only actual entries/exits write them) and can tolerate a few seconds of staleness — a strong fit for an in-memory cache updated by the `AvailabilityListener` mechanism from §36-37, read directly by the display board without touching the database at all.

---

# 43. Persistence and Caching for a Large-Scale Deployment

```text
Write path (rare, must be durable):
  EntryGate/ExitGate --> Ticket + Payment written in ONE transaction --> Relational DB

Read path (frequent, tolerates brief staleness):
  Display board / mobile app --> reads current availability counts --> In-memory cache
                                   (kept current by AvailabilityListener push updates, §37,
                                    NOT by querying the database on every display refresh)

This is the SAME "move frequent reads off the transactional database" principle §12-13
already established when justifying purpose-built allocator pools over a naive DB-counter design.
```

Keeping these two paths physically separate — a durable transactional store for the rare, high-stakes writes (tickets/payments), a fast in-memory cache for the frequent, low-stakes reads (availability counts) — is the same general storage-tiering principle this series has applied repeatedly (the Analytics Platform's hot/cold store split, the Logging guide's async ring buffer decoupling slow I/O from the fast path).

---

# 44. Class Diagram: Pricing and Allocation Strategies

```text
+------------------------+                    +------------------------+
|    PricingStrategy       |                    |     SpotAllocator        |
|   <<interface>>           |                    |   <<interface>>          |
|  computeFee(ticket, now)  |                    |  allocate(vehicle)       |
+-----------+--------------+                    |  release(spot)           |
     ^            ^                              +-----------+--------------+
     |            |                                   ^            ^
+---------+ +--------------------+           +----------------+ +------------------------+
| Hourly  | | VehicleTypeDiffer- |           | BestFit        | | NearestAvailable        |
| Pricing | | entiatedPricing    |           | Allocation     | | AllocationStrategy      |
| Strategy| | Strategy (wraps    |           | Strategy       | |                          |
|         | |  another strategy) |           +----------------+ +------------------------+
+---------+ +--------------------+
                                                                +------------------------+
                                                                | ObservableSpotAllocator |
                                                                | (Decorator, wraps a      |
                                                                |  concrete allocator, §37)|
                                                                +------------------------+
```

Both hierarchies share the exact same shape — a narrow interface, several interchangeable concrete strategies, and at least one **Decorator** that wraps another implementation of the same interface to layer on additional behavior (vehicle-type multipliers for pricing, availability notification for allocation) — which is precisely the kind of structural consistency that makes a codebase like this easy to extend once a developer has internalized the pattern once.

---

# 45. Capacity Estimation: Spots, Tickets, and Throughput

```text
Assume: a large multi-level garage with 5,000 total spots, averaging 3 full turnovers/spot/day
Total entries/day = 5,000 * 3                          = 15,000 entries/day
Peak entries/hour (assume 20% of daily volume in the peak hour) = 3,000 entries/hour = ~50/minute

Per-ticket record size (vehicle, spot, timestamps, fee) ≈ 200 bytes
Ticket storage/day = 15,000 * 200 bytes                = 3 MB/day -- trivially small for a relational DB

Free-spot pool memory: 5,000 ParkingSpot REFERENCES held across a handful of type-partitioned
  pools -- a few hundred KB at most, comfortably held in a single process's memory, confirming
  that even a very large single lot never approaches a scale requiring distributed allocator state.
```

The numbers here land in a genuinely different regime than this series' other, internet-scale guides (the Analytics Platform's millions of events/sec, the Rate Limiter's fleet-wide Redis coordination) — a single parking lot's absolute scale is modest even at its largest realistic size, which is precisely why multi-location aggregation (§40-41), not single-lot scale, is where this design's real distributed-systems complexity actually lives.

---

# 46. Full Worked Example: One Vehicle's Journey, Traced End to End

```text
1. A compact car arrives at Entry Gate 2: EntryGate.processEntry(Vehicle("XYZ-123", COMPACT))
2. BestFitAllocationStrategy.allocate(vehicle) checks compatibleTypesFor(COMPACT) = [COMPACT, LARGE]
     a. COMPACT pool has a free spot: C-042 -- pollFirst() returns it atomically (§34)
3. Ticket "T-9981" created: { vehicle, spot=C-042, entryTime=09:14:02 }, persisted FIRST (§20)
4. C-042.markOccupied() -- FreeState.assign() -> OccupiedState (§29-30)
5. ObservableSpotAllocator notifies listeners: "COMPACT count decreased to 41" -- display board updates (§36-37)
6. Physical gate opens, vehicle proceeds to spot C-042

--- 3 hours 20 minutes later ---

7. Vehicle arrives at Exit Gate 1, ticket "T-9981" scanned: ExitGate.processExit("T-9981")
8. HourlyPricingStrategy.computeFee(ticket, 12:34:02): 3h20m parked, 15-min grace period already
   elapsed, billable hours rounded UP to 4 -- fee = 4 * ratePerHour (§24)
9. PaymentProcessor.charge(vehicle, fee) -> SUCCESS
10. Ticket updated: exitTime=12:34:02, fee recorded, persisted
11. C-042.markFree() -- OccupiedState.release() -> FreeState (§29-30)
12. SpotAllocator.release(C-042) returns it to the COMPACT free pool -- available for the NEXT vehicle
13. ObservableSpotAllocator notifies listeners: "COMPACT count increased to 42" -- display board updates
14. Physical gate opens, vehicle departs
```

Every mechanism introduced by a follow-up question in this guide appears somewhere in this one vehicle's journey — the allocation strategy, the persist-before-transition ordering, the state machine, the Observer notification, and the pricing strategy are not independent, optional features, they are the actual steps one real entry-to-exit cycle passes through in this design.

---

# 47. Final Architecture Diagram

```text
                    +------------------+
Vehicle arrives --->|   Entry Gate      |
                    +--------+---------+
                             |
                    +--------v---------+
                    | ObservableSpotAlloc|--notifies-->  AvailabilityListeners (display board, app, metrics)
                    | (wraps BestFit/    |
                    |  NearestAvailable) |
                    +--------+---------+
                             |
                    +--------v---------+       +------------------------+
                    |     Ticket         |------>|   TicketRepository       |
                    | (State: SpotState) |       |   (relational DB)        |
                    +--------+---------+       +------------------------+
                             |
                     ... vehicle parks ...
                             |
                    +--------v---------+       +------------------------+
Vehicle returns --->|   Exit Gate        |------>|   PricingStrategy       |
                    +--------+---------+       |  (Hourly, wrapped by     |
                             |                  |   VehicleTypeDifferent.) |
                    +--------v---------+       +------------------------+
                    | Payment Processor  |
                    +------------------+
```

---

# 48. Design Patterns Used Throughout This Guide

- **Strategy** — `SpotAllocator` (§16-18: best-fit vs. nearest-available) and `PricingStrategy` (§23-24: hourly, flat-rate, vehicle-type-differentiated) are both swappable behaviors selected independently of the gate logic that uses them.
- **State** — `SpotState` (§29-30) encodes a parking spot's legal transitions directly into per-state objects, structurally preventing illegal operations rather than relying on scattered runtime checks.
- **Decorator** — `VehicleTypeDifferentiatedPricingStrategy` (§24) and `ObservableSpotAllocator` (§37) both wrap another implementation of the same interface to layer on additional behavior without modifying the wrapped implementation.
- **Observer** — `AvailabilityListener` (§36-37) lets any number of independent subscribers (display board, mobile app, metrics exporter) react to availability changes without the allocator needing to know who, or how many, are listening.
- **Facade** — `EntryGate`/`ExitGate` each hide allocation, persistence, pricing, and payment behind a single, simple entry point application code (or a physical gate controller) calls.

---

# 49. SOLID Principles Applied

- **Single Responsibility** — `SpotAllocator` only decides which spot to assign; `PricingStrategy` only computes a fee; `TicketRepository` only persists tickets — none of the three knows how to do the others' job, even though `EntryGate`/`ExitGate` compose all of them on every call.
- **Open/Closed** — adding a new vehicle type, spot type, pricing scheme, or allocation policy each means adding a new enum value or a new interface implementation, never modifying `EntryGate`/`ExitGate`'s own orchestration logic.
- **Liskov Substitution** — every `SpotState` implementation must honestly support the same three transition methods with the same contract (return a new legal state, or throw for an illegal one), so `ParkingSpot` can delegate to whichever state it currently holds without any special-casing.
- **Interface Segregation** — `PricingStrategy` exposes exactly one method, so a simple flat-rate implementation isn't forced to depend on grace-period or multiplier concepts only a more elaborate implementation needs.
- **Dependency Inversion** — `EntryGate` and `ExitGate` depend only on the `SpotAllocator`, `PricingStrategy`, and `TicketRepository` abstractions, never on concrete implementations, so any of the three can be swapped per deployment without touching gate logic.

---

# 50. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Scattering vehicle/spot-type if-checks across classes    Single canFit() method, exhaustive switch (§10, §13)
Linear-scanning every spot to find one that's free        Pre-partitioned free-spot pools per type (§15-16)
A boolean isOccupied flag with ad hoc if-checks           Explicit State pattern with legal transitions (§29-30)
Freeing a spot before payment succeeds at exit             Payment-then-free ordering (§25-26)
Transitioning spot state before the ticket is persisted    Persist-then-transition ordering (§20)
Check-then-act across two separate calls on a "thread-safe" Single atomic pollFirst() call, no separate
  collection, reintroducing a race anyway                    isEmpty() check (§32-34)
Polling a display board against the live database           Push-based Observer notifications + cache (§35-37, §43)
Routing every gate decision through a central multi-        Independent per-lot instances, async push (§40-41)
  location system
```

---

# 51. Testing Strategy

- **Allocation strategy tests** — verify `BestFitAllocationStrategy` prefers the tightest-fitting available spot type, and that `NearestAvailableAllocationStrategy` correctly prioritizes distance over fit-tightness; verify both correctly return empty when no compatible spot is free.
- **State machine tests** — assert every illegal transition (`OCCUPIED.assign()`, `FREE.release()`, `OUT_OF_SERVICE.assign()`) throws, and every legal transition produces the correct resulting state.
- **Pricing strategy tests** — verify `HourlyPricingStrategy`'s grace period and hour-rounding behavior at exact boundary values (e.g., exactly at the grace period's edge, exactly on an hour boundary), and verify `VehicleTypeDifferentiatedPricingStrategy` correctly delegates to and multiplies its wrapped strategy's result.
- **Concurrency tests** — spin up many concurrent simulated entry gates racing for a small, fixed pool of spots, and assert the total number of successful allocations never exceeds the actual number of spots that existed.
- **End-to-end lifecycle tests** — a full entry-to-exit trace (mirroring §46's worked example) asserting the ticket, spot state, availability count, and payment all end in mutually consistent states.

---

# 52. Suggested Future Enhancements

- **Reservations** — allowing a driver to reserve a specific spot (or spot type) in advance, requiring a new `RESERVED` state transition path already sketched in §29's state diagram but not fully implemented here.
- **Dynamic/surge pricing** — a `PricingStrategy` implementation that varies its rate based on current occupancy level, mirroring the Rate Limiter guide's own load-adaptive future-enhancement idea.
- **Electric vehicle charging spots** — a new `SpotType` with an associated charging-session duration and its own pricing dimension, layered on top of the existing type-compatibility and pricing infrastructure without modifying either.
- **License-plate recognition integration** — automating the vehicle-identity-capture step at entry (§19-20) via a camera/OCR pipeline, removing the need for a physical ticket at all for registered/subscribed vehicles.
- **Cross-lot availability search** — extending §40-41's central aggregation service into a driver-facing "find the nearest lot with a free spot" API across an entire multi-location chain.

---

# 53. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Model the core domain entities and explain how your design avoids scattering vehicle/spot-type checks across multiple classes. (§9-13)
2. Design spot allocation so it doesn't scan every spot on every request, and explain the best-fit vs. nearest-available tradeoff. (§14-18)
3. Walk through the exact entry and exit flows, including why certain operations must happen in a specific order. (§19-27)
4. Design the spot/ticket lifecycle so an already-occupied spot cannot structurally be assigned again. (§28-30)
5. Two entry gates race for the last free compact spot at the exact same instant. Diagnose the failure mode and fix it. (§32-34)
6. Design real-time availability reporting to a display board without polling the database on every refresh. (§35-37)
7. Extend this design from one lot to a multi-location chain reported on centrally, without making routine entry/exit depend on network connectivity to a central system. (§40-41)
8. A driver arrives at the exit without their ticket. Design the fallback flow. (§38-39)

---

# 54. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **make illegal states and illegal operations structurally impossible to express, rather than merely checking for them at runtime** — a `canFit()` method instead of scattered type checks (§10, §13), a `SpotState` object that simply doesn't offer an illegal transition rather than an `if` guarding against one (§29-30), a single atomic `pollFirst()` call instead of a check-then-act sequence that could race (§32-34). None of these are exotic techniques — they are the same discipline, applied consistently, of designing the data and its operations so that the compiler and the type system do as much of the correctness work as possible, leaving less for a developer (or a future maintainer) to have to remember to check by hand.

---

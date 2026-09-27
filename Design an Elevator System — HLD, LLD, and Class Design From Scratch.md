# Design an Elevator System — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Hall call (floor button) --> [ Elevator Controller ] --dispatches--> [ Elevator Car ]
Car call (inside button)  -->        |                                    |
                                [ Dispatch Strategy ]              [ State Machine ]
                                (FCFS, SCAN, Nearest-Car)      (Idle/MovingUp/MovingDown/DoorsOpen)
                                      |
                              [ Metrics: wait time, travel time ]
```

An elevator system looks like a solved, everyday problem, but a *system* built around it — one dispatching several cars across many floors, minimizing wait time under real request volume, remaining fair under sustained load, respecting capacity limits, and handling emergencies — is a genuinely rich object-oriented and scheduling-algorithm design exercise. This guide builds one from scratch: a clean domain model distinguishing hall calls from car calls, an explicit state machine for each car's lifecycle, the classic SCAN/LOOK dispatch algorithm and why naive first-come-first-served produces poor wait times, multi-elevator dispatch via a pluggable strategy, starvation prevention, capacity and emergency handling, and the metrics that actually distinguish a good dispatch algorithm from a bad one.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Explain precisely why naive first-come-first-served dispatch produces poor average wait times, and design the classic SCAN (elevator) algorithm that fixes it.
- Model an elevator's lifecycle as an explicit state machine, and distinguish hall calls from car calls in a way that correctly encodes *direction* as part of a request.
- Design multi-elevator dispatch as a pluggable strategy, and compare first-come-first-served, SCAN, and nearest-car heuristics on real tradeoffs, not just intuition.
- Diagnose and fix a starvation failure mode where a request can be repeatedly passed over in favor of newer ones.
- Design for capacity limits, door sensors, and an emergency-override mode that must take priority over normal dispatch.
- Choose and justify the metrics (average wait time, average travel time, worst-case wait) that actually distinguish dispatch algorithm quality.
- Apply SOLID principles and recognizable design patterns (State, Strategy, Observer) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design an elevator system" is a classic object-oriented design question precisely because its rules are familiar to everyone, which removes any excuse for hiding weak design behind domain complexity — every choice (state modeling, dispatch strategy, fairness) is fully visible and judged purely on its own merits. It also has a genuine algorithmic core (the SCAN/LOOK family of disk-scheduling-adjacent algorithms) and a genuine distributed-systems-flavored dimension once multiple cars must coordinate who answers which call — both of which reward a candidate who reaches for the right technique rather than a plausible-sounding guess. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Sealed interfaces and records model request/state precisely; a `PriorityQueue` gives an efficient basis for a car's own sorted stop list |
| Runnable artifact | HTML5 Canvas + vanilla JavaScript | Single-file, dependency-free, lets dispatch algorithms be watched and compared live in a browser |
| Dispatch algorithm | SCAN (elevator algorithm), plus FCFS and nearest-car for comparison | SCAN is the standard, well-understood algorithm real elevator controllers are built around; comparing it against naive alternatives is the whole point of this guide |
| Concurrency | Per-elevator request queue, one lock scope per car | Confines contention to the specific car a request is being added to, never a single building-wide lock |
| Metrics | Average wait time, average travel time, worst-case (starvation) wait | The three numbers that actually distinguish a good dispatch algorithm from a bad one, not just "does it look reasonable" |

---

# 5. Project Structure

```text
elevator-system/
├── src/main/java/com/example/elevator/
│   ├── domain/
│   │   └── Floor.java, Request.java, Direction.java, ElevatorState.java  // §10, §18
│   ├── car/
│   │   ├── ElevatorCar.java (State pattern)                              // §16
│   │   └── CarStateMachine.java                                          // §15-16
│   ├── dispatch/
│   │   ├── DispatchStrategy.java (Strategy interface)                    // §22
│   │   ├── FirstComeFirstServedStrategy.java                             // §26
│   │   ├── ScanDispatchStrategy.java                                     // §22
│   │   └── NearestCarStrategy.java                                       // §26
│   ├── controller/
│   │   ├── ElevatorController.java                                      // §24-26
│   │   └── FairnessAgingPolicy.java (starvation prevention)               // §28
│   ├── safety/
│   │   └── CapacityGuard.java, EmergencyOverride.java                    // §30, §35
│   └── metrics/
│       └── DispatchMetrics.java                                          // §37
├── src/test/java/com/example/elevator/
│   ├── ScanOrderingTest.java
│   ├── StarvationPreventionTest.java
│   └── ConcurrentHallCallTest.java
└── artifact/
    └── elevator-arena.html   -- the runnable simulator, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design an elevator system."*

Even for a system this familiar, the intentionally open prompt needs narrowing: how many elevators, and how many floors? Is the goal to minimize average wait time, worst-case wait time, or energy usage — these can genuinely conflict? Does the building have unusual traffic patterns (a lobby-heavy "up-peak" every morning) worth designing around specifically? Is there a weight/capacity limit to respect, and an emergency mode that must override normal dispatch? The answers reshape which parts of the design carry real weight, and asking them signals the difference between reciting "just use SCAN" and actually designing *this* building's system.

---

# 7. Functional Requirements

- **Accept hall calls** (a floor button pressed, requesting up or down) and **car calls** (a destination button pressed inside a car).
- **Dispatch an appropriate elevator** to answer each hall call, and move each car through its own queued stops in a sensible order.
- **Track and transition each elevator's state** correctly: idle, moving up, moving down, doors open, out of service.
- **Support multiple elevators** serving the same building, coordinating who answers which hall call.
- **Respect capacity limits**, refusing to add further passengers once a car is full.
- **Support an emergency/priority mode** (e.g., a fire alarm) that overrides normal dispatch entirely.
- **Track dispatch performance** (average wait time, average travel time) so different dispatch strategies can be compared.

---

# 8. Non-Functional Requirements

- **Low dispatch latency**: deciding which elevator answers a new hall call must be fast enough to feel instantaneous to a waiting passenger, even in a large building with many cars.
- **Fairness**: no request should be able to wait indefinitely while newer requests are repeatedly served first (starvation).
- **Correctness under concurrency**: hall calls arriving from multiple floors at nearly the same instant must never corrupt a car's queue or be silently dropped.
- **Safety**: a capacity-exceeding load, or an open-door condition, must never be silently ignored — these are hard constraints, not optimization targets.
- **Extensibility**: adding a new dispatch algorithm should require adding new code, not modifying the existing controller or car state machine.
- **Graceful degradation**: a single car going out of service must not stop the rest of the building's elevators from continuing to operate.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Floor** — a single level in the building, the unit both hall calls and car destinations are expressed in.
- **Request** — either a **hall call** (floor, direction) made from outside a car, or a **car call** (destination floor) made from inside one — a distinction the next few sections show is not cosmetic.
- **Elevator Car** — a single physical car, with a current floor, a current state, and its own queue of stops to serve.
- **Elevator Controller** — the building-wide coordinator that decides which car answers each new hall call.
- **Dispatch Strategy** — the pluggable policy the controller uses to make that decision.
- **Direction** — `UP` or `DOWN`, attached to a hall call specifically because a car already heading up shouldn't be dispatched to a passenger who wants to go down, even if it's momentarily closer.

---

# 10. Identifying the Core Domain Entities

```java
public enum Direction { UP, DOWN }
public enum ElevatorState { IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN, OUT_OF_SERVICE }

public record HallCall(int floor, Direction direction, Instant requestedAt) { }
public record CarCall(int destinationFloor) { }

public class ElevatorCar {
    private final String carId;
    private int currentFloor;
    private ElevatorState state;
    private final TreeSet<Integer> upStops = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>();
    private int currentLoad = 0;
    private final int capacity;
}
```

Splitting a car's pending stops into two separate sorted sets — `upStops` and `downStops` — rather than one flat list is the single modeling decision that makes the SCAN algorithm (§20-22) fall out almost for free: a car moving up only ever needs the *next-higher* entry in `upStops`, and a car moving down only ever needs the *next-lower* entry in `downStops`, both O(log n) operations on a `TreeSet`.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Hall/Car call ----->|   ElevatorController     |
                    |  dispatch(hallCall)      |
                    +-----------+--------------+
                                |
                    +-----------v--------------+
                    |    DispatchStrategy        |
                    |  FCFS | SCAN | NearestCar  |
                    +-----------+--------------+
                                |
                    +-----------v--------------+
                    |      ElevatorCar[]         |
                    |  state, upStops, downStops |
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |    CapacityGuard            |       |   DispatchMetrics       |
                    |    EmergencyOverride         |       |   wait/travel time      |
                    +------------------------+       +------------------------+
```

`ElevatorController` is deliberately the single place a hall call's *destination car* is decided — every car's own movement logic (§15-16) only ever needs to know its own queued stops, never anything about the other cars or how it was assigned this particular stop.

---

# 12. Follow-up Question 2 — "Why Not Just Send Each Elevator to Requests in the Order They Arrive?"

> **Interviewer:** *"The simplest possible dispatch: each elevator serves whichever request it received first, in arrival order. What's wrong with that?"*

Because arrival order has nothing to do with **physical proximity** — a car could be sitting on floor 2 when request A (floor 18) arrives, then request B (floor 3) arrives a moment later; serving A first means physically passing right by floor 3 twice (once on the way up, once again after eventually coming back down) instead of simply stopping there on the way. First-come-first-served optimizes for a property (fairness in arrival order) that has no relationship to what actually determines wait time: how much physical travel a given ordering forces the car to do.

---

# 13. Why a Naive FCFS Architecture Produces Poor Wait Times

```text
Car at floor 2. Requests arrive in this ORDER: floor 18 (A), floor 3 (B), floor 15 (C)

FCFS (strict arrival order):  2 -> 18 (serve A) -> 3 (serve B) -> 15 (serve C)
   Total floors traveled: |18-2| + |3-18| + |15-3| = 16 + 15 + 12 = 43 floors

SCAN (serve everything encountered along the current direction of travel, §20-22):
   2 -> 3 (serve B, encountered on the way up) -> 15 (serve C) -> 18 (serve A)
   Total floors traveled: |3-2| + |15-3| + |18-15| = 1 + 12 + 3 = 16 floors
```

This is precisely the same insight behind disk-arm scheduling (SCAN is literally borrowed from that domain) — reordering requests to match the server's *physical* traversal, rather than the *arrival* order they happened to be requested in, is what actually minimizes total travel, and therefore average wait time.

---

# 14. Follow-up Question 3 — "How Do You Model Elevator State So It Can't Accept an Illegal Transition?"

> **Interviewer:** *"What stops your code from, say, opening the doors while the car is still moving, or accepting a new destination while out of service?"*

By making a car's current phase an explicit, small, closed set of values with clearly defined legal transitions between them — exactly the same **State** pattern this series has used for a parking spot's lifecycle and a Tic-Tac-Toe game's phase — rather than a handful of independent boolean flags (`isMoving`, `isDoorOpen`) that any code, anywhere, could set into a combination that's physically nonsensical (doors open *and* moving, simultaneously).

---

# 15. Elevator Lifecycle as a State Machine

```text
IDLE ----assignStop()----> MOVING_UP / MOVING_DOWN ----arriveAtStop()----> DOORS_OPEN
DOORS_OPEN ----doorsTimeout() / closeButtonPressed()----> IDLE (or immediately re-dispatched if more stops queued)
ANY STATE ----takeOutOfService()----> OUT_OF_SERVICE ----bringBackInService()----> IDLE

Illegal transitions (structurally PREVENTED, not just discouraged by convention):
  MOVING_UP.openDoors()        -- doors cannot open while the car is in motion
  OUT_OF_SERVICE.assignStop()  -- a car taken out of service cannot be dispatched to a new stop
  DOORS_OPEN.assignStop()      -- (a genuinely subtle case, §16) -- a new stop is QUEUED, not IMMEDIATELY acted on
```

The `DOORS_OPEN.assignStop()` case is worth flagging early: a hall call arriving while a car's doors happen to be open at some other floor must be queued for later, never causing the car to suddenly slam its doors shut and move — this is exactly the kind of subtlety a state machine makes explicit and a scattered boolean-flag design tends to get quietly wrong.

---

# 16. Implementing Elevator State Transitions

```java
public interface CarState {
    CarState assignStop(ElevatorCar car, int floor);
    CarState arriveAtNextStop(ElevatorCar car);
    CarState doorsTimeout(ElevatorCar car);
}

public class IdleState implements CarState {
    @Override
    public CarState assignStop(ElevatorCar car, int floor) {
        car.enqueueStop(floor);
        return floor > car.currentFloor() ? new MovingUpState() : new MovingDownState();
    }
    @Override
    public CarState arriveAtNextStop(ElevatorCar car) { throw new IllegalStateException("Idle car has no destination to arrive at"); }
    @Override
    public CarState doorsTimeout(ElevatorCar car) { throw new IllegalStateException("Doors are not open"); }
}

public class MovingUpState implements CarState {
    @Override
    public CarState assignStop(ElevatorCar car, int floor) { car.enqueueStop(floor); return this; } // queued, no immediate transition
    @Override
    public CarState arriveAtNextStop(ElevatorCar car) { car.openDoors(); return new DoorsOpenState(); }
    @Override
    public CarState doorsTimeout(ElevatorCar car) { throw new IllegalStateException("Doors are not open"); }
}

public class ElevatorCar {
    private CarState state = new IdleState();
    public void requestStop(int floor) { this.state = state.assignStop(this, floor); }
    public void onArrival() { this.state = state.arriveAtNextStop(this); }
}
```

Each concrete `CarState` owns exactly the transitions legal from that specific phase — `ElevatorCar` itself never contains a single `if (currentState == ...)` check; it simply delegates every operation to whichever state object it currently holds, exactly the same delegation structure this series' Parking Lot and Tic-Tac-Toe guides both used for their own state machines.

---

# 17. Follow-up Question 4 — "Why Distinguish Hall Calls from Car Calls, and Why Does Direction Matter?"

> **Interviewer:** *"A passenger on floor 5 presses 'up.' A passenger already inside a car presses '12.' Aren't both just 'go to a floor'? Why model them differently?"*

Because a **hall call** carries information a **car call** structurally cannot: the requester's *intended direction*, known before they've even boarded a car. A car call ("take me to floor 12") has no ambiguity — the car is already committed to serving that specific passenger. But a hall call ("I'm on floor 5, I want to go up") must be matched only to a car that is *already heading up and hasn't passed floor 5 yet*, or one that's idle — dispatching a car heading *down* through floor 5 to answer an "going up" hall call would mean picking up a passenger who then has to ride down past their own floor before eventually going up, which is never what SCAN-style dispatch should do.

---

# 18. Hall Calls vs. Car Calls, and Why Direction Matters

```text
Car currently at floor 10, moving DOWN, with floor 5 still ahead of it (on the way down).

Hall call: floor 5, requesting UP     -- do NOT let this car answer it, even though floor 5 is
                                          directly in its path -- picking up an "UP" passenger while
                                          moving DOWN means they ride the WRONG WAY before their
                                          actual trip even starts.

Hall call: floor 5, requesting DOWN   -- this car CAN legitimately answer it -- it's already
                                          heading the correct direction and floor 5 is ahead of it.
```

This is precisely why `HallCall` (§10) carries a `Direction` field that `CarCall` structurally has no equivalent for — the dispatch algorithm's correctness depends on this distinction being available as data, not re-derived unreliably from context at decision time.

---

# 19. Implementing the Request Model

```java
public class ElevatorCar {
    // ... existing fields ...
    private Direction currentDirection; // null when IDLE

    public boolean canServe(HallCall call) {
        if (state instanceof OutOfServiceState) return false;
        if (currentDirection == null) return true; // idle, or doors open with no committed direction -- can serve anything

        boolean movingTowardCallFloor = currentDirection == Direction.UP
            ? call.floor() >= currentFloor
            : call.floor() <= currentFloor;
        boolean sameDirectionAsRequested = currentDirection == call.direction();
        return movingTowardCallFloor && sameDirectionAsRequested;
    }
}
```

`canServe` is the single method that encodes §17-18's entire rule — every dispatch strategy in this guide (§20-26) calls this method rather than re-implementing the direction-compatibility check itself, which is exactly what keeps the rule centralized and correctly enforced regardless of which strategy is currently active.

---

# 20. Follow-up Question 5 — "Design the Classic SCAN Algorithm — What Problem Does It Actually Fix, Precisely?"

> **Interviewer:** *"You've shown FCFS travels more floors than necessary. Design the actual algorithm real elevators use, and state its guarantee precisely."*

**SCAN** (also called the elevator algorithm, borrowed directly from disk-arm scheduling): a car continues moving in its **current direction**, serving every pending stop it encounters along the way, until no further stops remain ahead of it in that direction — only then does it reverse. This guarantees a car never backtracks past a pending stop it could have served on the way, which is precisely the inefficiency §12-13 identified in naive FCFS.

---

# 21. The SCAN (Elevator) Algorithm

```text
Car moving UP, upStops = {12, 15, 18}, currently at floor 9:
  Serve 12 (next-highest in upStops that's still ahead) -> serve 15 -> serve 18
  upStops now empty -> check downStops -- if non-empty, REVERSE direction and serve those, descending
  If downStops also empty -> transition to IDLE, awaiting the next assigned stop

At every step, the car only ever needs: "what's the NEXT stop in upStops that's >= currentFloor"
(a single TreeSet.ceiling() call, O(log n)) -- never a scan over the entire stop list.
```

The `TreeSet` split from §10 is what makes this genuinely cheap — `upStops.ceiling(currentFloor)` and `downStops.floor(currentFloor)` are the only two operations SCAN's per-step decision ever needs, each a single O(log n) call rather than any kind of linear search over pending stops.

---

# 22. Implementing SCAN Dispatch

```java
public interface DispatchStrategy {
    ElevatorCar selectCarFor(HallCall call, List<ElevatorCar> fleet);
}

public class ElevatorCar {
    private final TreeSet<Integer> upStops = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>();

    public void enqueueStop(int floor) {
        if (floor >= currentFloor) upStops.add(floor); else downStops.add(floor);
    }

    public OptionalInt nextStop() {
        if (currentDirection == Direction.UP) {
            Integer next = upStops.ceiling(currentFloor);
            if (next != null) return OptionalInt.of(next);
            return downStops.isEmpty() ? OptionalInt.empty() : OptionalInt.of(downStops.last()); // reverse
        } else {
            Integer next = downStops.floor(currentFloor);
            if (next != null) return OptionalInt.of(next);
            return upStops.isEmpty() ? OptionalInt.empty() : OptionalInt.of(upStops.first()); // reverse
        }
    }
}

public class ScanDispatchStrategy implements DispatchStrategy {
    @Override
    public ElevatorCar selectCarFor(HallCall call, List<ElevatorCar> fleet) {
        return fleet.stream()
            .filter(car -> car.canServe(call))          // direction-compatible, per §19
            .min(Comparator.comparingInt(car -> Math.abs(car.currentFloor() - call.floor())))
            .orElseGet(() -> nearestIdleOrReversingCar(call, fleet)); // fallback, §25-26
    }
}
```

`ScanDispatchStrategy` itself is a thin selection layer over each car's own SCAN-ordered stop list (§21) — the actual travel-minimizing behavior lives entirely inside `ElevatorCar.nextStop()`, which every dispatch strategy in this guide reuses unchanged, regardless of *which* car a given hall call gets assigned to.

---

# 23. Class Diagram: The Core Elevator Domain Model

```text
+------------------------+        +------------------------+
|       ElevatorCar         |------->|       CarState            |
|  upStops, downStops,     |       |    <<interface>>          |
|  requestStop(), onArrival()|      |  assignStop(), arrive()  |
+-----------+--------------+        +-----------+--------------+
                                          ^      ^      ^      ^
                                    +--------+ +--------+ +----------+ +----------------+
                                    | Idle   | | Moving | | Doors    | | OutOfService   |
                                    | State  | | Up/Down| | OpenState| | State          |
                                    +--------+ +--------+ +----------+ +----------------+

+------------------------+        +------------------------+
|      HallCall             |------->|     DispatchStrategy     |
|  floor, direction         |       |    <<interface>>          |
+------------------------+        |  selectCarFor(call,fleet)|
                                    +------------------------+
```

`ElevatorCar` is deliberately unaware of `DispatchStrategy` entirely — it only ever exposes `canServe`, `enqueueStop`, and `nextStop`, and the controller (§24-26) is the only component that ever asks a strategy to choose *among* several cars, which is precisely the separation that lets a new dispatch strategy be added without touching `ElevatorCar` at all.

---

# 24. Follow-up Question 6 — "With Multiple Elevators, How Do You Decide Which One Answers a Given Hall Call?"

> **Interviewer:** *"A building has six elevators. A hall call arrives. How do you decide which of the six actually gets dispatched?"*

Among every car that `canServe` the call (§19's direction-compatibility filter already narrows this), prefer the one that will reach the requesting floor **soonest** — in the common case, this reduces to simple physical distance, but a genuinely idle car and a car already moving toward the call in the right direction are both legitimate candidates, and the estimate needs to account for which one actually is.

---

# 25. Dispatching Among Multiple Elevators: The Nearest-Car Heuristic

```text
Hall call: floor 8, requesting UP

Car A: floor 3, IDLE                           -- estimated arrival: |8-3| = 5 floors of travel
Car B: floor 10, MOVING_DOWN, will reverse      -- estimated arrival: MUCH further -- must first
       once it clears its downStops                finish its current downward stops, THEN pass
                                                     back through floor 8 -- not simply |10-8|=2
Car C: floor 6, MOVING_UP, already correct dir  -- estimated arrival: |8-6| = 2 floors -- BEST candidate

Naive "closest by raw floor distance" would have picked Car B (only 2 floors away) -- but Car B is
moving the WRONG direction (§17-18) and must reverse first, making its TRUE arrival estimate far
worse than its raw distance suggests.
```

The correct estimate is never raw floor distance alone — it must account for direction compatibility (already filtered by `canServe`) and, for a car with existing queued stops, however many of those it must still clear before it could plausibly divert to the new call.

---

# 26. Implementing Multi-Elevator Dispatch as a Pluggable Strategy

```java
public class NearestCarStrategy implements DispatchStrategy {
    @Override
    public ElevatorCar selectCarFor(HallCall call, List<ElevatorCar> fleet) {
        return fleet.stream()
            .filter(ElevatorCar::isInService)
            .filter(car -> car.canServe(call))
            .min(Comparator.comparingInt(car -> estimatedArrivalCost(car, call)))
            .orElseThrow(() -> new NoAvailableCarException(call));
    }

    private int estimatedArrivalCost(ElevatorCar car, HallCall call) {
        if (car.isIdle()) return Math.abs(car.currentFloor() - call.floor());
        // a busy, direction-compatible car's cost includes every stop still ahead of it before this one
        return car.remainingStopsBeforeReaching(call.floor());
    }
}

public class FirstComeFirstServedStrategy implements DispatchStrategy {
    @Override
    public ElevatorCar selectCarFor(HallCall call, List<ElevatorCar> fleet) {
        return fleet.stream().filter(ElevatorCar::isInService).findFirst() // arrival order, no cost estimate at all
            .orElseThrow(() -> new NoAvailableCarException(call));
    }
}
```

`NearestCarStrategy` and `FirstComeFirstServedStrategy` implement the identical `DispatchStrategy` interface introduced in §22 — a textbook **Strategy** pattern application — which means a building operator can switch dispatch algorithms via configuration alone, with zero changes to `ElevatorController`'s own orchestration logic, and this guide's own interactive artifact (§1) lets these strategies be compared side by side on exactly this basis.

---

# 27. Follow-up Question 7 — "Nearest-Car Dispatch Can Starve a Request. How Do You Prevent That?"

> **Interviewer:** *"A request on a distant, awkward floor keeps losing out to newer, more conveniently-placed requests. It could wait far longer than any individual request 'deserves.' How do you prevent this?"*

By tracking **how long each hall call has already been waiting**, and letting that waiting time factor into dispatch cost, not just physical distance — a request that has waited unusually long should become effectively "closer" (in cost terms) than its raw distance suggests, until some car is finally compelled to answer it. This is the same general **aging** technique used in CPU and I/O scheduling to guarantee no request waits forever, applied here to hall-call dispatch instead.

---

# 28. Preventing Starvation: Aging and Fairness in Dispatch

```java
public class FairnessAgingPolicy {
    private static final Duration STARVATION_THRESHOLD = Duration.ofSeconds(45);
    private static final int STARVATION_PRIORITY_BONUS = 1000; // large enough to always win a cost comparison

    public int adjustedCost(int rawCost, HallCall call) {
        Duration waited = Duration.between(call.requestedAt(), Instant.now());
        if (waited.compareTo(STARVATION_THRESHOLD) > 0) {
            return rawCost - STARVATION_PRIORITY_BONUS; // effectively guarantees this call wins dispatch NOW
        }
        return rawCost;
    }
}
```

`NearestCarStrategy` (§26) would apply this adjustment before comparing costs across cars — the aging bonus is deliberately large enough to dominate any ordinary distance-based comparison once a request crosses the starvation threshold, converting "this request has waited unusually long" from an ignored fact into the single most important factor in the next dispatch decision.

---

# 29. Follow-up Question 8 — "How Do You Handle Capacity Limits So a Car Doesn't Accept a Hall Call It Physically Can't Fulfill?"

> **Interviewer:** *"A nearly-full car is dispatched to a hall call along its route, but there's no room left for the new passenger by the time it arrives. How do you prevent that?"*

By treating current load as a hard, checked precondition on **boarding**, not merely on dispatch — a car can legitimately be dispatched toward a hall call it currently has room for, and still arrive with insufficient room if it picked up unexpected additional passengers along the way (car calls don't carry weight information the controller can plan around in advance). The correct fix is a **door sensor / capacity guard** check at the moment of boarding, independent of whatever dispatch decision got the car there — the two checks protect against genuinely different failure windows.

---

# 30. Capacity Constraints and Door Sensor Handling

```java
public class CapacityGuard {
    public boolean canBoard(ElevatorCar car, int additionalWeight) {
        return car.currentLoad() + additionalWeight <= car.capacity();
    }

    public void onDoorSensorTriggered(ElevatorCar car) {
        // a physical obstruction (door sensor) reopens the doors regardless of the current
        // state machine's timer -- SAFETY overrides the normal DOORS_OPEN -> IDLE timeout
        car.reopenDoors();
    }
}
```

The door sensor callback is a good example of a safety concern that must be able to **interrupt** the normal state machine's timing — `reopenDoors()` doesn't wait for the state machine to reach a convenient point, it takes effect immediately, which is precisely why safety-critical checks like this are kept as a distinct, always-active concern rather than folded into the ordinary state transition logic §15-16 already established.

---

# 31. Follow-up Question 9 — "How Do You Stay Correct Under Concurrency When Several Hall Calls Arrive at Nearly the Same Instant?"

> **Interviewer:** *"Two hall calls, from different floors, arrive within milliseconds of each other. What stops the dispatch decision or a car's own stop queue from being corrupted?"*

By scoping any required lock to the **specific car** being modified, never to the whole building — dispatching two *different* hall calls to two *different* cars should never contend with each other at all, and even dispatching two hall calls to the *same* car only needs to briefly serialize that one car's `enqueueStop` calls, not block dispatch decisions for the rest of the fleet.

---

# 32. Concurrency: Thread-Safe Request Queue Per Elevator

```text
WITHOUT per-car locking:
  Hall call A dispatched to Car 1 --\
  Hall call B dispatched to Car 2 ---+-- if these share ONE global lock, B waits on A
                                          for NO REASON -- they don't touch the same car at all

WITH per-car locking (this guide's design):
  Car 1's enqueueStop() is synchronized on CAR 1's OWN monitor
  Car 2's enqueueStop() is synchronized on CAR 2's OWN monitor
  -- A and B proceed FULLY IN PARALLEL, since they never contend for the same lock
```

`ElevatorCar.enqueueStop` (§22) is the one operation that actually mutates shared state (`upStops`/`downStops`), so it's the one operation that needs synchronization — everything else (dispatch cost estimation, direction compatibility checks) only ever *reads* a car's current state, which a `TreeSet` snapshot or simple volatile reads can serve safely without any lock at all.

---

# 33. Class Diagram: Multi-Elevator Dispatch

```text
+------------------------+        +------------------------+
|   ElevatorController      |------->|    DispatchStrategy      |
|  onHallCall(call)         |       |    <<interface>>          |
+-----------+--------------+        +-----------+--------------+
            |                              ^      ^      ^
            v                        +----------+ +--------+ +----------------+
  +------------------------+          | FCFS     | | Scan   | | NearestCar     |
  |    FairnessAgingPolicy    |          | Strategy | |Strategy| | Strategy       |
  |  adjustedCost(cost,call)  |          +----------+ +--------+ +----------------+
  +------------------------+
            |
            v
  +------------------------+
  |      ElevatorCar[]        |
  |  (per-car lock scope,     |
  |   §32)                    |
  +------------------------+
```

`FairnessAgingPolicy` sits between the controller and whichever concrete strategy is active, adjusting the cost every strategy computes rather than being baked into any one of them individually — which means starvation prevention applies uniformly no matter which dispatch algorithm a building operator has currently selected.

---

# 34. Follow-up Question 10 — "How Do You Handle a Fire Alarm or Emergency Mode That Must Override Normal Dispatch?"

> **Interviewer:** *"A fire alarm triggers. Every elevator must immediately abandon its current queue and return to the ground floor, doors open, no longer accepting hall calls. How does that fit into your design?"*

By treating emergency mode as a **priority override layered on top of** the normal state machine and dispatch pipeline, not a special case threaded through every existing method — `ElevatorController` checks for an active emergency **before** ever consulting a `DispatchStrategy`, and each car's own state machine gains one additional transition (any state directly to a dedicated emergency-recall behavior) that takes precedence over whatever it was doing.

---

# 35. Emergency Override and Priority Modes

```java
public class ElevatorController {
    private volatile boolean emergencyActive = false;

    public void onHallCall(HallCall call) {
        if (emergencyActive) return; // normal hall calls are simply IGNORED during an emergency
        ElevatorCar chosen = dispatchStrategy.selectCarFor(call, fleet);
        chosen.requestStop(call.floor());
    }

    public void activateEmergency() {
        emergencyActive = true;
        for (ElevatorCar car : fleet) {
            car.clearAllQueuedStops();   // abandon whatever it was doing
            car.forceRecallToGroundFloor(); // a dedicated transition, bypassing normal dispatch entirely
        }
    }
}
```

Checking `emergencyActive` as the very first line of `onHallCall` — rather than threading an emergency check through `DispatchStrategy`, `CapacityGuard`, or any car's own state machine — is what keeps emergency handling a single, auditable override point, instead of a concern scattered across every other component that would each need to remember to check for it.

---

# 36. Follow-up Question 11 — "How Do You Evaluate and Compare Dispatch Algorithms — What Metrics Actually Matter?"

> **Interviewer:** *"You've now got three candidate dispatch strategies. How do you actually decide which one is better, rather than just guessing from intuition?"*

Three metrics, each capturing a genuinely different failure mode: **average wait time** (how long, on average, a passenger waits from hall call to pickup — the headline number most people mean by "elevator performance"), **average travel time** (how long a trip takes once boarded — a strategy that minimizes wait time by packing cars full can quietly worsen this), and **worst-case wait time** (the single longest any request ever waited — the number that specifically exposes starvation, invisible in an average that many quick, easy requests can dilute).

---

# 37. Evaluating Dispatch Algorithms: Wait Time, Travel Time, and Starvation

```text
                     Avg. wait time    Avg. travel time    Worst-case wait
FCFS                 Poor (§12-13)     Reasonable           Poor (arrival order offers
                                                              no protection against a run
                                                              of badly-placed requests)
SCAN (single car)     Good              Reasonable           Reasonable (bounded by one
                                                              full sweep of the building)
NearestCar            Best (multi-car)  Good                 CAN starve without §27-28's
                                                              aging fix -- must be paired
                                                              with fairness, not assumed safe alone
```

No single metric tells the whole story — a strategy that wins on average wait time while quietly producing occasional multi-minute worst-case waits (exactly nearest-car's failure mode without aging) is not actually the better strategy for a real building, which is why this guide's own interactive artifact (§1) surfaces all three numbers side by side rather than picking one "the" score.

---

# 38. Follow-up Question 12 — "How Would This Scale to a Very Tall, Very High-Traffic Building?"

> **Interviewer:** *"A 60-story office tower, with a sharp 'up-peak' every morning as everyone arrives around 9am. Does this design still work well, unmodified?"*

Not unmodified — a single pool of general-purpose elevators serving every floor becomes genuinely inefficient once traffic is this concentrated and directional, and real high-rise buildings address this with structural changes beyond dispatch-algorithm tuning: **zoning** (dedicating specific elevator banks to specific floor ranges, so a car serving floors 30-45 never wastes a trip stopping at floor 4), and **double-deck elevators** (two stacked cabins sharing one shaft, serving odd and even floors simultaneously, doubling a single shaft's effective throughput) — both are structural answers to a traffic pattern that a smarter dispatch algorithm alone cannot fully solve.

---

# 39. Scaling to High-Rise, High-Traffic Buildings: Zoning and Double-Deck Elevators

```text
Un-zoned (this guide's core design):  6 elevators, EACH serving ALL 60 floors --
  every car potentially stops at every floor, diluting effective capacity during a sharp up-peak

Zoned:  elevators 1-2 serve floors 1-20; elevators 3-4 serve floors 21-40; elevators 5-6 serve
  floors 41-60 -- each car's EFFECTIVE dispatch problem shrinks to a 20-floor building, and SCAN's
  own per-sweep efficiency (§21) improves proportionally, since there's simply less building for
  each car to physically traverse per trip.
```

Zoning doesn't require a new dispatch *algorithm* at all — it's implemented as a simple additional filter inside `canServe` (§19): a car only ever reports itself able to serve a hall call whose floor falls within its assigned zone, and every mechanism this guide already built (SCAN, nearest-car, aging) continues operating correctly, just within a narrower floor range per car.

---

# 40. Capacity Estimation: Requests per Minute and Dispatch Latency

```text
Assume: a 20-floor office building, 4 elevators, peak morning traffic generating one hall call
every 3 seconds building-wide (a busy but realistic up-peak)

Hall calls/minute = 60 / 3                              = 20 calls/minute
Per-call dispatch decision cost: filtering 4 cars + one TreeSet lookup each -- microseconds,
  utterly negligible compared to the SECONDS a physical elevator takes to travel between floors

This confirms dispatch DECISION latency is never the bottleneck at any realistic building scale --
  the actual constraint is always PHYSICAL (car speed, door-cycle time, number of cars), never the
  CPU cost of deciding which car to send, which is precisely why capacity planning for a real
  elevator system is a mechanical/traffic-engineering problem first, a software-scaling problem second.
```

This is a genuinely different capacity story than this series' other guides (a rate limiter's Redis throughput, an analytics platform's ingestion volume) — the software layer here is comfortably fast at any realistic scale, and the real constraint this guide's own §38-39 already addressed (zoning, double-deck cars) is a physical, not computational, one.

---

# 41. Full Worked Example: One Hall Call, Traced End to End

```text
1. A passenger on floor 8 presses "UP" -- HallCall(floor=8, direction=UP, requestedAt=now) created
2. ElevatorController.onHallCall(call) checks emergencyActive -- false, proceeds normally (§35)
3. FairnessAgingPolicy.adjustedCost() checked for every candidate car -- no request has aged past
   the starvation threshold yet, so raw costs are used unmodified this time (§27-28)
4. NearestCarStrategy.selectCarFor(call, fleet):
     a. Filters to cars where canServe(call) is true (§19) -- Car C (floor 6, MOVING_UP) qualifies;
        Car B (floor 10, MOVING_DOWN) does NOT (wrong direction)
     b. estimatedArrivalCost: Car C = 2 floors ahead; Car A (idle, floor 3) = 5 floors -- Car C wins
5. Car C's enqueueStop(8) called, synchronized on Car C's own lock only (§32) -- Car A and Car B's
   dispatch state are entirely untouched, no contention
6. Car C's own SCAN ordering (§21) naturally includes floor 8 in its next-stop sequence, since it's
   directly ahead in its current direction of travel
7. Car C arrives at floor 8: CarState.arriveAtNextStop() transitions MovingUpState -> DoorsOpenState
   (§16), doors open, the passenger boards (subject to CapacityGuard.canBoard(), §30)
8. DispatchMetrics records this call's wait time (requestedAt to arrival) for later comparison
   against whichever OTHER dispatch strategy might be evaluated next (§36-37)
```

Every mechanism introduced by a follow-up question in this guide appears somewhere in this one hall call's journey — direction compatibility, cost estimation, fairness aging, per-car locking, and SCAN ordering are not independent, optional features, they are the actual steps one real hall call passes through in this design.

---

# 42. Final Architecture Diagram

```text
                    +------------------------+
Hall/Car call ----->|   ElevatorController     |----(emergency check first, §34-35)
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |  FairnessAgingPolicy       |------>|    DispatchStrategy      |
                    |  (wraps cost, §27-28)      |       |  FCFS | SCAN | NearestCar|
                    +------------------------+       +-----------+--------------+
                                                                   |
                                +----------------------------------+----------------------------------+
                                v                                  v                                  v
                    +------------------+              +------------------+              +------------------+
                    |   ElevatorCar 1    |              |   ElevatorCar 2    |              |   ElevatorCar N    |
                    | upStops/downStops  |              | (own lock, §32)    |              |                    |
                    | CarState machine   |              +------------------+              +------------------+
                    +---------+----------+
                              |
                    +---------v----------+       +------------------------+
                    |   CapacityGuard      |       |    DispatchMetrics       |
                    +------------------+       +------------------------+
```

---

# 43. Design Patterns Used Throughout This Guide

- **State** — `CarState` (§15-16) encodes exactly which transitions are legal from an elevator's current phase, structurally preventing illegal operations (opening doors mid-motion, dispatching an out-of-service car) rather than relying on scattered runtime checks.
- **Strategy** — `DispatchStrategy` (§22-26: FCFS, SCAN, nearest-car) is a single swappable interface behind which every dispatch algorithm lives, interchangeable via configuration alone.
- **Decorator** — `FairnessAgingPolicy` (§28) wraps whichever cost calculation a concrete strategy produces, layering starvation prevention on top without any strategy needing to implement it itself.
- **Facade** — `ElevatorController` hides emergency checking, fairness adjustment, and strategy selection behind a single `onHallCall` entry point.
- **Observer** (implicit) — `DispatchMetrics` reacts to completed trips (wait time, travel time) without the controller or any car needing to know how many downstream consumers exist.

---

# 44. SOLID Principles Applied

- **Single Responsibility** — `ElevatorCar` only tracks its own state/queue/movement; `DispatchStrategy` only decides which car answers a call; `CapacityGuard` only enforces load limits — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new dispatch algorithm (or a new safety check) means implementing one interface, never modifying `ElevatorController` or `ElevatorCar`.
- **Liskov Substitution** — every `CarState` implementation must honestly support the same three transition methods with the same contract (a new legal state, or a thrown exception for an illegal one), so `ElevatorCar` can delegate to whichever state it currently holds without special-casing.
- **Interface Segregation** — `DispatchStrategy` exposes exactly one method, so a trivial FCFS implementation isn't forced to depend on cost-estimation or aging concepts only more elaborate strategies actually need.
- **Dependency Inversion** — `ElevatorController` depends only on the `DispatchStrategy` abstraction, never on a concrete algorithm directly, so the active strategy can be swapped per deployment without touching controller logic.

---

# 45. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Dispatching by raw arrival order (FCFS)                   SCAN reorders by physical travel (§12-13, §20-22)
A boolean isMoving flag with scattered if-checks           Explicit State pattern with legal transitions (§15-16)
Treating a hall call the same as a car call                Hall calls carry direction; car calls don't (§17-19)
Picking the CLOSEST car by raw floor distance alone         Direction-compatible + queue-aware cost (§24-26)
Nearest-car dispatch with no fairness mechanism             Aging bonus prevents starvation (§27-28)
Checking capacity only at dispatch time, not boarding       CapacityGuard checked at the boarding moment (§29-30)
One global lock across the whole fleet for every dispatch  Per-car lock scope, contention-free (§31-32)
Threading an emergency check through every component        Single override point in the controller (§34-35)
```

---

# 46. Testing Strategy

- **SCAN ordering tests** — enqueue a mix of up and down stops onto one car and assert `nextStop()` always returns the correct next stop for the car's current direction, reversing only once the current direction's stops are exhausted.
- **State machine tests** — assert every illegal transition (opening doors while moving, dispatching an out-of-service car) throws, and every legal transition produces the correct resulting state.
- **Direction-compatibility tests** — assert `canServe` correctly rejects a hall call whose requested direction doesn't match a busy car's current direction, and correctly accepts it for an idle car.
- **Starvation prevention tests** — synthetically age one request past the threshold and assert it wins dispatch over a closer, more recently-arrived competing request.
- **Concurrent hall-call tests** — fire many simulated hall calls at different cars simultaneously and assert no car's stop queue is corrupted, while confirming dispatch to *different* cars proceeds without contending on a shared lock.

---

# 47. Suggested Future Enhancements

- **Destination dispatch (hall call with intended floor entered up front)** — replacing simple up/down hall-call buttons with a keypad that captures the destination floor *before* boarding, letting the controller group passengers headed to similar floors onto the same car far more precisely than direction alone allows.
- **Predictive pre-positioning** — during a well-understood recurring traffic pattern (a morning up-peak), proactively distributing idle cars toward the lobby *before* the rush begins, rather than reacting only once calls start arriving.
- **Reinforcement-learning-based dispatch** — the *elevator group control problem* is a genuine, decades-studied optimization problem in operations research, with published work applying reinforcement learning (state: car positions and queued calls; action: which car answers each hall call; reward: negative wait time) to outperform hand-tuned heuristics under complex, non-stationary traffic patterns — a legitimate next step, though a materially larger undertaking than this guide's SCAN/nearest-car strategies, requiring a full building traffic simulator to train against safely before ever touching a real building.
- **Energy-aware dispatch** — incorporating idle-car parking and selective standby power-down into the cost function itself, trading a small amount of average wait time for meaningfully lower energy consumption during low-traffic periods.
- **Cross-building analytics** — aggregating `DispatchMetrics` (§37) across many buildings running the same system, to validate whether a given dispatch strategy's advantages generalize beyond one specific building's traffic pattern.

---

# 48. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Explain precisely why first-come-first-served dispatch produces worse average wait times than SCAN, with a concrete example. (§12-13, §20-22)
2. Model an elevator's lifecycle so illegal transitions (doors opening mid-motion) are structurally impossible, not merely checked for. (§14-16)
3. Explain why a hall call and a car call need different data, and design the direction-compatibility rule this enables. (§17-19)
4. Design dispatch across multiple elevators, including why raw floor distance alone is the wrong cost function. (§24-26)
5. Nearest-car dispatch can starve a request. Diagnose the failure mode and design the fix. (§27-28)
6. Two hall calls arrive at nearly the same instant. Design the concurrency model that keeps dispatch correct without unnecessary contention. (§31-32)
7. A fire alarm triggers. Design how your system overrides normal dispatch, and explain why the check belongs where you put it. (§34-35)
8. What metrics would you actually use to decide whether a new dispatch algorithm is better than the current one, and why isn't one number enough? (§36-37)

---

# 49. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **optimize for physical reality, not for the order information happens to arrive in** — SCAN reorders stops by travel distance instead of arrival order (§12-13, §20-22); nearest-car dispatch estimates true arrival cost instead of raw floor distance (§24-26); aging corrects for a request's actual waited time instead of treating every dispatch decision as freshly independent (§27-28). Recognizing that the *naive* ordering of events (arrival order, raw distance, an isolated decision) is rarely the ordering that minimizes real-world cost is the transferable skill this guide is really teaching — the exact same lesson disk-arm scheduling taught operating-systems design decades before an elevator ever borrowed it.

---

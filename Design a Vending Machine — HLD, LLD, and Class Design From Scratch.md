# Design a Vending Machine — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Customer selects a slot --> [ Machine State Machine ] --awaits payment--> [ Payment Strategy ]
                                     |                                          |
                              IDLE / HAS_FUNDS / DISPENSING              coins | bills | card
                                     |
                          [ Inventory: atomic check-and-decrement ] --dispense--> [ Change Maker ]
                                                                                        |
                                                                          greedy, OR dynamic
                                                                          programming, and why
                                                                          that distinction is real
```

A vending machine looks like the simplest possible interview question — until the follow-ups start. This guide builds one from scratch: the machine's operation modeled as an explicit state machine (the textbook example for the State design pattern), pluggable payment methods, the greedy change-making algorithm and the precise, provable case where it stops being optimal, the dynamic-programming fix, atomic inventory dispensing that survives a stuck button or a double-press, refund handling, and scaling from one machine to a monitored fleet.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Model a vending machine's operation as an explicit state machine, and explain why a handful of independent boolean flags is the wrong way to represent it.
- Design a pluggable payment strategy supporting coins, bills, and card, without any payment-method-specific logic leaking into the machine's core state transitions.
- Implement the greedy change-making algorithm, and — critically — prove precisely when it stops producing the minimum number of coins, with a concrete counterexample.
- Implement the correct, general dynamic-programming fix for minimum-coin change, and explain why it's needed for a non-canonical denomination set even though greedy works fine for most real currencies.
- Design atomic inventory dispensing that can't be tricked into releasing two items for one payment by a double-press or a stuck sensor.
- Design refund handling, low-stock detection, and the scaling story from a single machine to a remotely-monitored fleet.
- Apply SOLID principles and recognizable design patterns (State, Strategy) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design a vending machine" endures as an interview question specifically because it's the canonical teaching example for the State pattern — a machine that must never dispense on an unpaid balance, or accept a second payment while already dispensing, is a small enough domain to fully implement in an interview, while still exposing whether a candidate reaches for explicit states or a tangle of boolean flags. It also hides a genuinely surprising algorithmic trap in an operation everyone assumes is trivial: making change — most candidates confidently write greedy change-making and never realize it can be *provably wrong* for the exact denomination set the interviewer chose on purpose. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Sealed interfaces model machine state and payment methods precisely; records keep inventory/change snapshots immutable |
| Runnable artifact | HTML5 Canvas + vanilla JavaScript | Single-file, dependency-free, lets the change-making comparison be watched and verified live in a browser |
| Change-making (canonical currency) | Greedy algorithm | Provably optimal for canonical denomination systems (like real-world currency), and far cheaper to compute |
| Change-making (general case) | Dynamic programming | The only approach that's provably optimal for *any* denomination set, including deliberately non-canonical ones |
| Inventory dispensing | Atomic check-and-decrement | The same "collapse check-and-act into one operation" principle this series has applied to every other shared-resource guide |
| Fleet monitoring | Periodic telemetry push per machine | Low-stock and fault alerts reach an operator without requiring anyone to physically visit every machine |

---

# 5. Project Structure

```text
vending-machine/
├── src/main/java/com/example/vending/
│   ├── domain/
│   │   └── Slot.java, Product.java, Money.java, Coin.java              // §10
│   ├── machine/
│   │   ├── MachineState.java (State pattern)                          // §13-14
│   │   └── VendingMachine.java                                        // §14
│   ├── payment/
│   │   ├── PaymentStrategy.java (Strategy interface)                  // §17
│   │   └── CoinPaymentStrategy.java, CardPaymentStrategy.java          // §18
│   ├── change/
│   │   ├── GreedyChangeMaker.java                                     // §21
│   │   └── DynamicProgrammingChangeMaker.java                         // §25
│   ├── inventory/
│   │   └── InventoryManager.java (atomic dispense)                    // §30
│   └── fleet/
│       └── TelemetryReporter.java                                     // §36
├── src/test/java/com/example/vending/
│   ├── GreedyChangeCounterexampleTest.java
│   ├── DoubleDispenseTest.java
│   └── MachineStateTransitionTest.java
└── artifact/
    └── vending-arena.html   -- the runnable simulator, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design a vending machine."*

Even for a machine everyone has personally used, the intentionally open prompt needs narrowing: how many product slots, and does each hold a single product type or a mix? What payment methods — coins and bills only, or card too? Does the machine need to compute change at all, or is it exact-change-only? Is this a single standalone machine, or does the design need to account for a whole fleet, remotely monitored for restocking? The answers reshape which parts of the design carry real weight, and asking them signals the difference between reciting "it has states" and actually designing *this* machine.

---

# 7. Functional Requirements

- **Display available products** with their price and current stock, per slot.
- **Accept payment** via multiple methods (coins, bills, card) up to or exceeding a selected product's price.
- **Dispense the selected product** only once sufficient funds have been received, and only if it's actually in stock.
- **Return change** for any overpayment, using the minimum number of coins/bills possible.
- **Support cancellation**, refunding whatever has been inserted so far if the customer changes their mind before dispensing.
- **Track inventory per slot**, detecting and reporting low-stock or out-of-stock conditions.
- **Support a fleet of machines**, each reporting its own inventory and fault status remotely.

---

# 8. Non-Functional Requirements

- **Correctness above all**: the machine must never dispense a product without having received sufficient payment, and never dispense two products for one payment.
- **Minimum-coin change**: whatever change algorithm is used must be provably correct for the machine's actual denomination set, not merely "usually right."
- **Atomicity under mechanical failure**: a stuck sensor or a rapid double button-press must never cause double-dispensing or an inventory count drifting out of sync with reality.
- **Graceful degradation on low change reserves**: the machine should switch to exact-change-only rather than dispense a product it then can't correctly give change for.
- **Bounded fleet monitoring latency**: an operator should learn of a low-stock or faulted machine within a reasonably short window, not only when someone happens to visit it.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Slot** — a single dispensing position, holding one product type, a price, and a current stock count.
- **Product** — the item a slot dispenses, with a name and a price.
- **Money** — an amount tendered or owed, always reasoned about as an integer count of the smallest currency unit (cents), never a floating-point dollar amount.
- **Payment Method** — coins, bills, or card — a pluggable way of contributing toward a selected product's price.
- **Machine State** — the machine's current operational phase (idle, awaiting more funds, dispensing, out of service) — an explicit, closed set of values, not a handful of independent flags.
- **Change Maker** — the algorithm that decides which coins/bills to return for a given overpayment amount.

---

# 10. Identifying the Core Domain Entities

```java
public record Product(String name, int priceCents) { }

public class Slot {
    private final String slotId;
    private final Product product;
    private int stockCount;

    public boolean isInStock() { return stockCount > 0; }
}

public record Coin(int valueCents) { } // e.g. 1, 5, 10, 25 -- or a deliberately NON-canonical set, §22-23

public class Balance {
    private int insertedCents = 0;
    public void add(int cents) { insertedCents += cents; }
    public int cents() { return insertedCents; }
}
```

Reasoning about every amount as an integer count of cents — never a floating-point dollar value — is a small but load-bearing detail: floating-point rounding error in a system that literally exists to handle exact money would be a real, embarrassing correctness bug, not a cosmetic one.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Customer selects  -->|      VendingMachine       |
slot, inserts money  |  (State pattern, §13-14) |
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |    PaymentStrategy         |------>|      Balance             |
                    |  coins | bills | card      |       +------------------------+
                    +------------------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |    InventoryManager        |------>|    ChangeMaker           |
                    |  atomic check-and-decrement|       |  greedy OR DP, §19-25    |
                    +------------------------+       +------------------------+

                    +------------------------+
Fleet of machines --->|   TelemetryReporter      |----(low-stock/fault alerts, §36)
                    +------------------------+
```

Every mechanism this guide builds sits along one linear path — select, pay, dispense, change — with `VendingMachine`'s own state machine as the single orchestrator deciding which of these steps is even legal to attempt at any given moment.

---

# 12. Follow-up Question 2 — "Why Not Just Use a Few Boolean Flags for the Machine's Status?"

> **Interviewer:** *"`hasSelectedProduct`, `hasSufficientFunds`, `isDispensing` — three booleans. What's wrong with tracking status that way?"*

Independent booleans can be set into combinations that are physically nonsensical — `isDispensing = true` while `hasSelectedProduct = false`, for instance, describes a machine dispensing nothing in particular, a state that should be structurally impossible, not merely "shouldn't happen if the code is written carefully." The fix, exactly as this series has applied repeatedly (a parking spot's lifecycle, an elevator car's lifecycle, a seat's booking lifecycle), is to make the machine's phase a single, explicit, closed set of values with clearly defined legal transitions between them.

---

# 13. Modeling the Machine as an Explicit State Machine

```text
IDLE ----selectProduct()----> AWAITING_PAYMENT
AWAITING_PAYMENT ----insertPayment(), balance still short----> AWAITING_PAYMENT (unchanged)
AWAITING_PAYMENT ----insertPayment(), balance now sufficient----> DISPENSING
DISPENSING ----dispenseComplete()----> IDLE (change returned as part of this same transition)
AWAITING_PAYMENT ----cancel()----> IDLE (full refund)
ANY STATE ----faultDetected()----> OUT_OF_SERVICE

Illegal transitions (structurally PREVENTED, not just discouraged by convention):
  IDLE.insertPayment()        -- no product selected yet, nothing to pay toward
  DISPENSING.selectProduct()  -- already mid-dispense, a new selection mid-flight is nonsensical
  OUT_OF_SERVICE.selectProduct() -- a faulted machine accepts no new transactions at all
```

This is the exact same State-pattern discipline this series has used for a parking spot, an elevator car, and a booked seat — a small, explicit set of phases, each exposing only the operations that are legal from it, with the state object itself (not scattered `if` checks) deciding what happens next.

---

# 14. Implementing the State Pattern for Machine Operation

```java
public interface MachineState {
    MachineState selectProduct(VendingMachine machine, Slot slot);
    MachineState insertPayment(VendingMachine machine, int cents);
    MachineState cancel(VendingMachine machine);
}

public class IdleState implements MachineState {
    @Override
    public MachineState selectProduct(VendingMachine machine, Slot slot) {
        if (!slot.isInStock()) throw new IllegalStateException("Slot is out of stock");
        machine.setSelectedSlot(slot);
        return new AwaitingPaymentState();
    }
    @Override
    public MachineState insertPayment(VendingMachine machine, int cents) {
        throw new IllegalStateException("Select a product before inserting payment");
    }
    @Override
    public MachineState cancel(VendingMachine machine) { return this; } // nothing to cancel
}

public class AwaitingPaymentState implements MachineState {
    @Override
    public MachineState selectProduct(VendingMachine machine, Slot slot) {
        throw new IllegalStateException("A product is already selected -- cancel first");
    }
    @Override
    public MachineState insertPayment(VendingMachine machine, int cents) {
        machine.balance().add(cents);
        if (machine.balance().cents() >= machine.selectedSlot().product().priceCents()) {
            return new DispensingState();
        }
        return this; // still short -- stay in this same state, awaiting more
    }
    @Override
    public MachineState cancel(VendingMachine machine) {
        machine.refund(machine.balance().cents());
        return new IdleState();
    }
}

public class VendingMachine {
    private MachineState state = new IdleState();
    public void selectProduct(Slot slot) { this.state = state.selectProduct(this, slot); }
    public void insertPayment(int cents) { this.state = state.insertPayment(this, cents); }
}
```

`VendingMachine` itself contains no `if (currentState == ...)` check anywhere — it simply delegates every operation to whichever state object it currently holds, exactly the same delegation structure this series has used consistently, which is what makes each state's legal operations fully self-contained and independently testable.

---

# 15. Class Diagram: The Vending Machine State Machine

```text
+------------------------+        +------------------------+
|     VendingMachine        |------->|      MachineState         |
|  selectProduct(),         |       |    <<interface>>          |
|  insertPayment(),         |       |  selectProduct(),         |
|  cancel()                 |       |  insertPayment(), cancel()|
+-----------+--------------+        +-----------+--------------+
            |                              ^      ^      ^      ^
            v                        +----------+ +----------+ +--------+ +----------------+
+------------------------+          | Idle     | | Awaiting | | Dispen | | OutOfService   |
|        Balance            |          | State    | | Payment  | | -sing  | | State          |
+------------------------+          +----------+ | State    | | State  | +----------------+
                                                     +----------+ +--------+
```

Every state transition method returns the *next* state object rather than mutating a bare enum field in place — this is what makes each transition's legality a compile-time-visible property of which interface method a given state class chooses to implement meaningfully versus throw from, rather than a runtime check scattered across `VendingMachine` itself.

---

# 16. Follow-up Question 3 — "How Do You Support Multiple Payment Methods — Coins, Bills, Card — Cleanly?"

> **Interviewer:** *"Coins and bills contribute a fixed amount instantly. A card payment requires an external authorization round-trip. How do you support both without the machine's state logic needing to know the difference?"*

By keeping `MachineState.insertPayment` agnostic to *how* a given amount was obtained — a `PaymentStrategy` is responsible for producing a validated cents amount (instantly, for coins/bills, or after an async authorization, for card), and only *once that amount is known* does it get handed to the exact same `insertPayment` call every other payment method uses. The state machine never needs a `PaymentMethod` parameter or a conditional branch at all.

---

# 17. Payment as a Pluggable Strategy

```java
public interface PaymentStrategy {
    void submitPayment(VendingMachine machine, PaymentRequest request);
}
```

Every payment method — instant or asynchronous — implements this identical interface, exactly mirroring the Strategy pattern this series has used for AI difficulty, dispatch algorithms, and pricing schemes throughout: swapping in a new payment method is a configuration change, never a change to `VendingMachine`'s own state transitions.

---

# 18. Implementing Coin/Bill and Card Payment Strategies

```java
public class CoinBillPaymentStrategy implements PaymentStrategy {
    @Override
    public void submitPayment(VendingMachine machine, PaymentRequest request) {
        machine.insertPayment(request.cents()); // instant -- the physical coin/bill acceptor already validated it
    }
}

public class CardPaymentStrategy implements PaymentStrategy {
    private final CardAuthorizer authorizer;

    @Override
    public void submitPayment(VendingMachine machine, PaymentRequest request) {
        authorizer.authorizeAsync(request.cardToken(), request.cents(), result -> {
            if (result.approved()) {
                machine.insertPayment(request.cents()); // arrives LATER, but through the SAME call
            } else {
                machine.reportPaymentFailure(result.reason());
            }
        });
    }
}
```

Both strategies eventually call the exact same `machine.insertPayment(cents)` — the *timing* differs completely (instant versus an asynchronous callback), but the *state machine's* own view of "payment arrived" is identical either way, which is precisely why `MachineState` never needed to change at all to add card support.

---

# 19. Follow-up Question 4 — "How Do You Compute Change — Walk Through the Algorithm"

> **Interviewer:** *"A customer pays $2.00 for a $1.65 item. Walk through exactly how the machine decides which coins to return for the 35 cents owed."*

The standard approach is **greedy**: repeatedly take the largest available denomination that doesn't exceed the remaining amount owed, subtract it, and repeat until the remaining amount reaches zero — for common currency denominations, this reliably produces the minimum possible number of coins, and it's simple enough to compute instantly with no search involved at all.

---

# 20. The Greedy Change-Making Algorithm

```text
Owed: 35 cents. Available denominations: 25, 10, 5, 1 (a real, canonical currency system)

Take 25 (largest <= 35)   -> remaining 10, coins used: [25]
Take 10 (largest <= 10)   -> remaining 0,  coins used: [25, 10]
DONE -- 2 coins, which IS the true minimum possible for 35 cents in this denomination system
```

Greedy's appeal is real: it's a single pass, no backtracking, and for the denomination systems most real currencies actually use, it happens to coincide with the true optimum — which is exactly why so many vending-machine implementations ship with greedy and never discover the problem §22-23 describes, because their currency's denominations happen to be forgiving.

---

# 21. Implementing Greedy Change-Making

```java
public class GreedyChangeMaker implements ChangeMaker {
    private final int[] denominationsDescending; // e.g. {25, 10, 5, 1}, sorted LARGEST FIRST

    @Override
    public List<Integer> makeChange(int owedCents) {
        List<Integer> coinsUsed = new ArrayList<>();
        int remaining = owedCents;
        for (int denomination : denominationsDescending) {
            while (remaining >= denomination) {
                coinsUsed.add(denomination);
                remaining -= denomination;
            }
        }
        if (remaining != 0) throw new IllegalStateException("Cannot make exact change with available denominations");
        return coinsUsed;
    }
}
```

Nothing about this implementation is wrong on its own terms — it correctly computes *a* valid way to make change, using as many of the largest denomination as possible before moving to the next. Whether that's the *minimum possible* number of coins is a separate question entirely, and §22-23 is where the answer stops being "obviously yes."

---

# 22. Follow-up Question 5 — "Does Greedy Always Give the Minimum Number of Coins? Prove It or Break It."

> **Interviewer:** *"You've implemented greedy change-making. Is it always optimal? If not, give me a concrete denomination set and target amount where it fails."*

It is not always optimal, and this is provable with a small, concrete counterexample rather than a hand-wave: for the denomination set `{1, 3, 4}` and a target of **6 cents**, greedy takes 4 first (the largest denomination not exceeding 6), leaving 2, which then forces two more 1-cent coins — three coins total (`4 + 1 + 1`). But `3 + 3` also sums to 6, using only **two** coins — greedy's own largest-first rule actively prevented it from ever considering this better answer, because taking a 4 first made reaching an even split by 3s impossible.

---

# 23. Why Greedy Fails for Non-Canonical Denominations

```text
Denominations: {1, 3, 4}.  Target: 6 cents.

Greedy:  take 4 (largest <= 6)  -> remaining 2
         take 1 (largest <= 2)  -> remaining 1
         take 1 (largest <= 1)  -> remaining 0
         RESULT: [4, 1, 1] -- 3 coins

Optimal: [3, 3] -- 2 coins -- greedy NEVER EVEN CONSIDERS this combination, because its
         first, irreversible choice (take the 4) already used up value that a 3-3 split needed.

This is NOT a contrived edge case in the abstract sense -- it's a precise, provable
mathematical fact about denomination sets that lack the "canonical" property most real-world
currencies happen to have (each denomination divides evenly enough into the next that greedy's
locally-best choice never forecloses a globally-better one). {1,3,4} genuinely lacks this
property, and greedy is provably suboptimal for it, not just occasionally unlucky.
```

Most real currencies (US cents: 1, 5, 10, 25; most world currencies similarly) happen to be canonical, which is precisely why greedy change-making has such a good reputation in practice — the algorithm isn't "usually right by luck," it's "exactly right for a well-behaved class of denomination sets," and the failure only surfaces once a denomination set falls outside that class, which does happen in some real currencies and in any custom token/points system a designer might invent without realizing the property matters.

---

# 24. The Correct Fix: Dynamic Programming for Minimum-Coin Change

```text
Idea: build up the answer for EVERY amount from 0 to the target, using previously-solved
smaller amounts -- never make an irreversible greedy commitment before checking every option.

minCoins[0] = 0 coins (trivially, zero owed needs zero coins)
For amount = 1 to target:
    minCoins[amount] = min over every denomination d <= amount of ( 1 + minCoins[amount - d] )
    -- i.e., "what's the best result if THIS coin is the last one used, for every possible last coin"

For {1,3,4}, target=6:
    minCoins[1]=1, minCoins[2]=2, minCoins[3]=1 (a single 3), minCoins[4]=1 (a single 4)
    minCoins[5]=min(1+minCoins[4], 1+minCoins[2], 1+minCoins[1]) = min(2,3,2) = 2  (a 4 plus a 1)
    minCoins[6]=min(1+minCoins[5], 1+minCoins[3], 1+minCoins[2]) = min(3, 2, 3) = 2  -- CORRECTLY finds 3+3
```

Dynamic programming is correct precisely because it never commits to a first choice before knowing its downstream consequences — every amount's optimal answer is computed from *already-optimal* answers to every smaller amount, which is what lets it discover the `3+3` combination greedy's own irrevocable first move ruled out.

---

# 25. Implementing DP-Based Change-Making

```java
public class DynamicProgrammingChangeMaker implements ChangeMaker {
    private final int[] denominations;

    @Override
    public List<Integer> makeChange(int owedCents) {
        int[] minCoins = new int[owedCents + 1];
        int[] lastCoinUsed = new int[owedCents + 1];
        Arrays.fill(minCoins, Integer.MAX_VALUE);
        minCoins[0] = 0;

        for (int amount = 1; amount <= owedCents; amount++) {
            for (int denomination : denominations) {
                if (denomination <= amount && minCoins[amount - denomination] != Integer.MAX_VALUE
                        && minCoins[amount - denomination] + 1 < minCoins[amount]) {
                    minCoins[amount] = minCoins[amount - denomination] + 1;
                    lastCoinUsed[amount] = denomination;
                }
            }
        }
        if (minCoins[owedCents] == Integer.MAX_VALUE) {
            throw new IllegalStateException("Cannot make exact change with available denominations");
        }

        List<Integer> coinsUsed = new ArrayList<>();
        int remaining = owedCents;
        while (remaining > 0) {
            coinsUsed.add(lastCoinUsed[remaining]);
            remaining -= lastCoinUsed[remaining];
        }
        return coinsUsed;
    }
}
```

`lastCoinUsed` is what lets this implementation *reconstruct* the actual optimal combination of coins, not merely report the minimum count — without tracking it, the algorithm would know "2 coins is possible" but have no way to say which two, which isn't useful for a machine that actually has to physically dispense specific coins.

---

# 26. Follow-up Question 6 — "What Happens If the Machine Runs Low on Specific Coin Denominations for Making Change?"

> **Interviewer:** *"Your change-making algorithm assumes unlimited supply of every denomination. A real machine's coin reserves run low. What changes?"*

Both `ChangeMaker` implementations need to account for **finite, currently-available quantities** per denomination, not just an unlimited abstract set — this changes the algorithm from "find the minimum coins from an infinite supply" to "find the minimum coins from *this specific, finite, currently-remaining inventory*," and when no valid combination exists at all (the machine has run out of small denominations entirely), the machine must refuse to complete transactions that would require change it cannot actually provide.

---

# 27. Exact-Change-Only Mode and Change Reserve Management

```java
public class InventoryAwareChangeMaker implements ChangeMaker {
    private final Map<Integer, Integer> availableCountByDenomination; // denomination -> remaining count
    private final ChangeMaker delegate; // wraps either greedy or DP, §21 or §25

    @Override
    public List<Integer> makeChange(int owedCents) {
        List<Integer> proposed = delegate.makeChange(owedCents);
        Map<Integer, Long> requiredCounts = proposed.stream()
            .collect(Collectors.groupingBy(c -> c, Collectors.counting()));

        for (Map.Entry<Integer, Long> entry : requiredCounts.entrySet()) {
            if (availableCountByDenomination.getOrDefault(entry.getKey(), 0) < entry.getValue()) {
                throw new InsufficientChangeException("Cannot make change -- switch to exact-change-only mode");
            }
        }
        proposed.forEach(coin -> availableCountByDenomination.merge(coin, -1, Integer::sum));
        return proposed;
    }
}
```

Catching `InsufficientChangeException` at the machine's own display layer, and switching to a visible "EXACT CHANGE ONLY" mode, is the correct user-facing response — it's a far better outcome than either silently dispensing wrong change or accepting a payment the machine then cannot correctly settle.

---

# 28. Follow-up Question 7 — "A Stuck Sensor or a Rapid Double Button-Press — How Do You Avoid Dispensing the Same Item Twice for One Payment?"

> **Interviewer:** *"A customer's button-press bounces (a common mechanical reality) and registers twice in quick succession. What stops the machine from dispensing two items but only charging for one?"*

This is precisely the same check-then-act race this series has diagnosed repeatedly for a parking spot and a booked seat — a naive "check stock is positive, then decrement" sequence has a window between the two steps where a second, nearly-simultaneous dispense request can also pass the check before the first request's decrement takes effect. The fix is identical in shape too: collapse the check and the decrement into a single **atomic** operation.

---

# 29. Concurrency: Atomic Check-and-Decrement for Inventory

```text
WITHOUT atomicity:
  Request A: read stockCount (=1)  -- sees "in stock"
  Request B: read stockCount (=1)  -- ALSO sees "in stock", before A's decrement lands
  Request A: stockCount = 0  -- decrements
  Request B: stockCount = -1 -- decrements AGAIN, past zero -- TWO items dispensed for ONE actual unit in stock

WITH atomicity (single compare-and-decrement operation):
  Request A: atomically checks stockCount > 0 AND decrements, IN ONE STEP -- succeeds, stockCount now 0
  Request B: atomically checks stockCount > 0 -- FALSE, since it's already 0 -- correctly REJECTED
```

The fix requires exactly the same discipline this series has applied to every other shared, contended resource: the read that decides "is this still available" and the write that commits to using it must happen as one indivisible operation, never as two separate steps with a window in between for another request to interleave.

---

# 30. Implementing Atomic Dispense

```java
public class InventoryManager {
    private final Map<String, AtomicInteger> stockBySlot = new ConcurrentHashMap<>();

    public boolean tryDispense(String slotId) {
        AtomicInteger stock = stockBySlot.get(slotId);
        // updateAndGet is ATOMIC: the read-check-decrement happens as one operation, no window to race in
        int[] resultHolder = new int[1];
        stock.updateAndGet(current -> {
            if (current > 0) { resultHolder[0] = 1; return current - 1; }
            resultHolder[0] = 0;
            return current; // unchanged -- nothing to dispense
        });
        return resultHolder[0] == 1;
    }
}
```

`AtomicInteger.updateAndGet` is doing the same job here that `SELECT ... FOR UPDATE` and a version-column compare-and-swap did for BookMyShow's seat booking (this series' own companion guide) — a single, indivisible read-modify-write operation, closing the exact same class of race this time for a physical item count instead of a seat's state.

---

# 31. Follow-up Question 8 — "A User Inserts Money, Then Cancels or the Machine Detects a Jam. How Do You Handle Refunds Correctly?"

> **Interviewer:** *"A customer inserts $2, then presses cancel before the item dispenses. Or the machine detects a mechanical jam mid-dispense. What happens to their money in each case?"*

Both cases route through the exact same refund mechanism — `AwaitingPaymentState.cancel` (§14) already returns the full inserted balance — but a mid-dispense jam is a genuinely different, higher-stakes case: the machine must determine whether the item **actually** left the slot (via a sensor) before deciding whether to refund, dispense again, or flag itself `OUT_OF_SERVICE` for a technician, since refunding *and* having already dispensed the item would be a real loss for the machine's operator.

---

# 32. Refund Handling and Partial-Payment Edge Cases

```java
public class DispensingState implements MachineState {
    @Override
    public MachineState onDispenseSensorResult(VendingMachine machine, boolean itemDetectedInTray) {
        int changeOwed = machine.balance().cents() - machine.selectedSlot().product().priceCents();
        if (itemDetectedInTray) {
            machine.releaseChange(machine.changeMaker().makeChange(changeOwed));
            machine.inventoryManager().confirmDispensed(machine.selectedSlot().slotId());
            return new IdleState();
        }
        // sensor did NOT detect the item -- a genuine jam. Do NOT assume either outcome silently.
        machine.flagForTechnician("Dispense sensor did not confirm item release");
        machine.refund(machine.balance().cents()); // refund the FULL amount, not just the change
        return new OutOfServiceState();
    }
}
```

Refunding the *entire* inserted amount (not merely the change owed) when the dispense sensor fails to confirm release is the conservative, correct default — the alternative (assuming the item dispensed successfully despite no sensor confirmation) risks charging a customer for an item they never actually received, which is a worse failure mode than a machine going briefly out of service for a technician to check.

---

# 33. Follow-up Question 9 — "How Do You Track and Alert on Low or Out-of-Stock Inventory Across Many Slots?"

> **Interviewer:** *"A machine has thirty slots. How does an operator know which ones need restocking without physically checking each one?"*

Each slot's `InventoryManager` already knows its own current count after every dispense (§30) — the natural extension is to compare that count against a configured low-stock threshold on every change, and emit an event the moment a slot crosses it, rather than requiring anyone to poll every slot's count on a schedule.

---

# 34. Inventory Management and Low-Stock Detection

```java
public class InventoryManager {
    private final Map<String, AtomicInteger> stockBySlot = new ConcurrentHashMap<>();
    private final Map<String, Integer> lowStockThresholdBySlot;
    private final List<LowStockListener> listeners;

    public boolean tryDispense(String slotId) {
        boolean dispensed = /* atomic check-and-decrement, §30 */ true;
        if (dispensed) {
            int remaining = stockBySlot.get(slotId).get();
            int threshold = lowStockThresholdBySlot.getOrDefault(slotId, 5);
            if (remaining <= threshold) {
                listeners.forEach(l -> l.onLowStock(slotId, remaining));
            }
        }
        return dispensed;
    }
}
```

Firing the low-stock event as a direct consequence of the *same* dispense operation that changed the count (rather than a separate periodic scan) means the alert is as fresh as the data itself — an operator learns about a slot crossing its threshold the moment it actually happens, with zero polling delay.

---

# 35. Follow-up Question 10 — "How Do You Scale This to a Fleet of Machines — Remote Monitoring, Restocking Routes?"

> **Interviewer:** *"A vending operator manages a thousand machines across a city. How does the design extend from one machine to a whole monitored fleet?"*

Each machine already emits low-stock and fault events locally (§33-34) — the fleet-scale extension is simply to have each machine **push** those same events to a central telemetry service over the network, rather than requiring a technician to physically visit every machine to discover its status. The core machine logic itself needs zero changes; telemetry is a listener attached to events the machine was already producing.

---

# 36. Fleet Telemetry and Remote Monitoring

```java
public class TelemetryReporter implements LowStockListener, FaultListener {
    private final String machineId;
    private final TelemetryClient telemetryClient; // sends to a central fleet-monitoring service

    @Override
    public void onLowStock(String slotId, int remaining) {
        telemetryClient.send(new LowStockEvent(machineId, slotId, remaining, Instant.now()));
    }

    @Override
    public void onFault(String reason) {
        telemetryClient.send(new FaultEvent(machineId, reason, Instant.now()));
    }
}
```

`TelemetryReporter` is simply one more subscriber to events the machine already emits — exactly the same Observer-pattern payoff this series has used consistently (a Snake game's dropped-tick counter, an elevator's alerting engine): adding fleet-wide monitoring requires zero changes to any machine's own core dispensing or state-machine logic, only a new listener attached to events already being fired.

---

# 37. Class Diagram: Full Vending Machine System

```text
+------------------------+        +------------------------+
|     VendingMachine        |------->|      MachineState         |
+-----------+--------------+        +------------------------+
            |
   +--------+--------+--------+
   v                 v         v
+----------+ +----------------+ +------------------------+
| Payment  | | InventoryManager| |     ChangeMaker          |
| Strategy |  | (atomic, §30)   | |  Greedy | DP, §21 / §25 |
+----------+ +--------+--------+ +------------------------+
                       |
                       v
             +------------------------+
             |   TelemetryReporter       |
             |   (Observer, §36)         |
             +------------------------+
```

Tracing this diagram is the entire system: the machine's own state governs what's legal, payment and inventory each operate independently behind their own pluggable interfaces, and telemetry sits entirely off to the side, observing without ever participating in the actual dispensing decision.

---

# 38. Follow-up Question 11 — "How Do You Support Promotional Pricing or Dynamic Pricing Per Product?"

> **Interviewer:** *"A product is on a temporary discount, or prices rise slightly for a premium slot. How does that fit in without complicating the state machine or dispensing logic?"*

Exactly the same answer this series has given for every other pricing question (the Parking Lot guide's vehicle-type differentiation, BookMyShow's tiered pricing): pricing stays **orthogonal** to the state machine entirely — a `PricingStrategy` is consulted only when displaying a price or finalizing what's owed, never touching machine state, inventory, or dispensing logic at all.

---

# 39. Pluggable Pricing and Promotions

```java
public interface PricingStrategy {
    int currentPriceCents(Product product);
}

public class PromotionalPricingStrategy implements PricingStrategy {
    private final PricingStrategy base; // wraps another strategy, same Decorator this series uses repeatedly
    private final int discountCents;

    @Override
    public int currentPriceCents(Product product) {
        return Math.max(0, base.currentPriceCents(product) - discountCents);
    }
}
```

`PromotionalPricingStrategy` wraps another `PricingStrategy` rather than reimplementing base pricing — the exact same compositional technique this series has used for pricing in every prior guide, letting a promotion be layered on top of any base pricing scheme without duplicating it.

---

# 40. Capacity Estimation: Transaction Throughput and Inventory Sync

```text
Assume: a single machine serves at most 1 transaction/minute in practice (mechanical dispense
        time dominates), across a fleet of 1,000 machines nationwide

Peak fleet-wide transaction rate ≈ 1,000 transactions/minute ≈ 17/second -- trivially small
  for any modern telemetry backend to ingest; a single machine's own dispensing logic is
  NEVER the bottleneck (unlike this series' higher-throughput guides), since a physical
  mechanical dispense cycle is orders of magnitude slower than any software decision involved

Telemetry payload: a low-stock/fault event is a few hundred bytes; even continuous fleet-wide
  telemetry at this rate is a negligible bandwidth cost, meaning fleet-scale monitoring is
  entirely a DATA MODELING and ALERTING-LOGIC problem here, not a raw-throughput one.
```

This is a deliberately different capacity story than this series' internet-scale guides (the Analytics Platform's millions of events/sec, BookMyShow's flash-sale spikes) — a vending machine's own mechanical dispensing rate caps throughput so low that software capacity is essentially never the actual constraint, which is worth naming explicitly rather than pretending every system in this series faces the same scaling pressure.

---

# 41. Full Worked Example: One Purchase, Traced End to End

```text
1. Customer selects slot B4 ($1.65 item): IdleState.selectProduct() -> AwaitingPaymentState (§13-14)
2. Customer inserts a $1 coin, then a $1 coin: insertPayment(100) called twice
     a. After the first: balance=100 cents, still short of 165 -- state stays AwaitingPaymentState
     b. After the second: balance=200 cents, now >= 165 -- transitions to DispensingState
3. DispensingState calls InventoryManager.tryDispense("B4") -- atomic check-and-decrement (§29-30)
     a. Succeeds: stock was 3, now 2 -- above the low-stock threshold, no alert fires
4. Dispense sensor confirms the item left the tray (§32): onDispenseSensorResult(true)
     a. changeOwed = 200 - 165 = 35 cents
     b. ChangeMaker.makeChange(35) -- e.g. DP-based, correctly returns the minimum coins
        possible from CURRENTLY AVAILABLE reserves (§27)
     c. Change released, state transitions back to IdleState
5. Suppose this was the slot's LAST unit before restock: stock now crosses the low-stock
   threshold -- LowStockListener fires, TelemetryReporter pushes an event to the central
   fleet-monitoring service (§34-36), entirely independent of anything the customer sees
```

Every mechanism this guide introduced via a follow-up question appears somewhere in this one purchase's trace — the state machine, atomic dispensing, correct change-making, and fleet telemetry are not independent, optional features, they are the actual steps one real transaction passes through in this design.

---

# 42. Final Architecture Diagram

```text
                    +------------------------+
Customer -------->  |     VendingMachine        |
                    |  (State pattern, §13-14) |
                    +-----------+--------------+
                                |
                +---------------+----------------+
                v                                  v
    +------------------------+          +------------------------+
    |    PaymentStrategy         |          |    InventoryManager      |
    |  coins | bills | card      |          |    (atomic, §29-30)      |
    +------------------------+          +-----------+--------------+
                                                       |
                                          +-------------v-------------+
                                          |       ChangeMaker            |
                                          |  Greedy | DP, per §21 / §25 |
                                          +------------------------+
                                                       |
                                          +-------------v-------------+
                                          |     TelemetryReporter        |
                                          |    (Observer) --> Fleet      |
                                          |    Monitoring Service        |
                                          +------------------------+
```

---

# 43. Design Patterns Used Throughout This Guide

- **State** — `MachineState` (§13-14) encodes exactly which operations are legal from the machine's current phase, structurally preventing illegal operations (dispensing without sufficient funds, accepting a new selection mid-dispense) rather than relying on scattered runtime checks.
- **Strategy** — `PaymentStrategy` (§17-18), `ChangeMaker` (§21, §25), and `PricingStrategy` (§39) each let a policy vary independently of the code that invokes it.
- **Decorator** — `InventoryAwareChangeMaker` (§27) and `PromotionalPricingStrategy` (§39) both wrap another implementation of the same interface to layer on additional behavior without modifying what they wrap.
- **Observer** — `TelemetryReporter` (§36) reacts to low-stock and fault events without the machine's own core logic needing to know how many downstream listeners exist or what they do with the notification.
- **Facade** — `VendingMachine` hides state transitions, payment, inventory, and change-making behind the few simple methods a physical control panel actually calls.

---

# 44. SOLID Principles Applied

- **Single Responsibility** — `InventoryManager` only tracks stock and dispenses atomically; `ChangeMaker` only computes change; `PaymentStrategy` only validates and reports a tendered amount — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new payment method, change-making algorithm, or pricing scheme each means implementing one interface, never modifying `VendingMachine`'s own state transitions.
- **Liskov Substitution** — every `ChangeMaker` implementation must honestly return a valid combination summing to the requested amount (or throw if genuinely impossible), so the machine can swap greedy for DP without any change to how the result is used.
- **Interface Segregation** — `PaymentStrategy` exposes exactly one method, so a trivial coin/bill implementation isn't forced to depend on card-authorization concepts only the card strategy actually needs.
- **Dependency Inversion** — `VendingMachine` depends on `PaymentStrategy`, `ChangeMaker`, and `InventoryManager` as abstractions, never on concrete implementations directly, so any of them can be swapped per deployment without touching the state machine itself.

---

# 45. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
A handful of independent boolean flags for machine status  Explicit State pattern with legal transitions (§12-14)
Reasoning about money as a floating-point dollar amount      Integer cents, always (§10)
Assuming greedy change-making is always optimal              Prove it, and know when DP is required (§22-25)
Checking stock, then decrementing, as two separate steps     Atomic check-and-decrement (§28-30)
Refunding only the change owed after a dispense-sensor fault  Refund the FULL inserted amount (§31-32)
Polling every slot's stock count on a schedule                Fire low-stock events as a direct consequence
                                                                of the dispense that caused them (§33-34)
Coupling promotional pricing into dispensing/state logic       Pricing kept fully orthogonal (§38-39)
```

---

# 46. Testing Strategy

- **State transition tests** — assert every illegal transition (inserting payment with no product selected, selecting a new product mid-dispense) throws, and every legal transition produces the correct resulting state.
- **Greedy-counterexample tests** — assert `GreedyChangeMaker` produces a *suboptimal* result for the `{1,3,4}`/6-cent case specifically, confirming the guide's own claim is reproducible in code, not just on paper.
- **DP-optimality tests** — assert `DynamicProgrammingChangeMaker` produces the true minimum coin count across a range of denomination sets, including both canonical and deliberately non-canonical ones.
- **Atomic dispense tests** — fire many concurrent `tryDispense` calls against a slot with a small fixed stock count and assert the number of successful dispenses never exceeds the actual stock available.
- **Refund tests** — simulate a dispense-sensor failure and assert the full inserted amount (not merely the change owed) is refunded, and the machine correctly transitions to `OutOfServiceState`.

---

# 47. Suggested Future Enhancements

- **Multi-item purchases** — allowing a customer to select several products before paying once, extending the state machine with a genuine "building an order" phase rather than one item per transaction.
- **Loyalty/points-based payment** — a new `PaymentStrategy` implementation redeeming accumulated points instead of currency, requiring zero changes to the state machine itself.
- **Predictive restocking routes** — using historical dispense-rate telemetry per machine to schedule restocking visits proactively, before a slot actually crosses its low-stock threshold, rather than reactively.
- **Cashless-only configurations** — a deployment variant skipping `ChangeMaker` entirely for machines that only ever accept card payment, demonstrating how cleanly an entire subsystem can be omitted when a specific deployment doesn't need it.
- **Tamper and fraud detection** — flagging unusual patterns (rapid repeated dispense attempts on one slot, coin-mechanism anomalies suggestive of counterfeit coins) as a specialized `FaultListener`, reusing the exact same telemetry pipeline built for ordinary low-stock and jam alerts.

---

# 48. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Model the machine's operation as an explicit state machine, and explain why a few independent boolean flags is the wrong approach. (§12-14)
2. Design pluggable payment support for coins, bills, and card, without the state machine needing to know which method is in use. (§16-18)
3. Implement greedy change-making, then prove — with a concrete counterexample — that it isn't always optimal. (§19-23)
4. Design and implement the correct general-case fix for minimum-coin change. (§24-25)
5. A stuck sensor or a rapid double button-press could dispense two items for one payment. Diagnose the race and design the fix. (§28-30)
6. A dispense-sensor fault occurs mid-transaction. Design exactly what happens to the customer's money. (§31-32)
7. Extend the design from one machine to a remotely-monitored fleet of a thousand, without changing any machine's own core logic. (§35-36)
8. Estimate this system's actual throughput bottleneck, and explain why it differs from most other systems in this series. (§40)

---

# 49. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **an operation that looks obviously correct is only as correct as the proof behind it, not the confidence behind it** — a state machine is correct because illegal transitions are structurally unrepresentable, not because the code "looks careful" (§12-14); greedy change-making is correct only for a specific, checkable property of the denomination set, not simply because it "seems reasonable" (§22-25); atomic dispensing is correct because the check and the decrement are one indivisible operation, not because a double-press "probably won't happen" (§28-30). Recognizing which of your own design's correctness claims actually have a proof behind them — and which are merely untested assumptions wearing the appearance of one — is the transferable skill this guide is really teaching.

---

# Design an ATM System — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Card inserted --> [ PIN Verification, retry-limited ] --> [ Select Transaction ]
                                                                   |
                                                          [ Reserve funds ] --dispense cash-->
                                                                   |                    |
                                                          [ Confirm debit ]      [ Cassettes: bounded
                                                                   |               inventory per note ]
                                        Network fails HERE? --> [ Reconciliation, later ]
```

An ATM looks like a simple cash-dispensing box, until the two questions every real deployment has to answer correctly: what happens when the network call confirming a debit times out *after* the cash has already physically left the machine, and how do you dispense an exact amount when each denomination's cassette holds only a limited, countable number of notes. This guide builds one from scratch: the session modeled as an explicit state machine, PIN retry-limit lockout, denomination-constrained cash dispensing (a genuinely harder variant of the classic change-making problem), the reserve-then-confirm pattern that prevents dispensing cash without a corresponding debit ever taking hold, background reconciliation for the ambiguous case where confirmation itself fails, concurrency-safe balance debiting across simultaneous withdrawal attempts, and graceful degradation when the network disappears mid-transaction.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Model an ATM session as an explicit state machine, including PIN retry-limit lockout and card retention.
- Design cash dispensing under a genuinely harder constraint than ordinary change-making: a fixed, countable number of notes per denomination, and correctly detect when an exact amount is undispensable even though total cash is sufficient.
- Diagnose the classic "cash dispensed, debit unconfirmed" failure and design the reserve-then-confirm pattern that prevents a network timeout from ever producing free money or a lost debit.
- Design background reconciliation for the case where even the confirmation step's own outcome is ambiguous.
- Design concurrency-safe balance debiting so two near-simultaneous withdrawal attempts on the same account can never both succeed past the available balance.
- Design fail-safe defaults for a mid-transaction network outage.
- Apply SOLID principles and recognizable design patterns (State, Strategy) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design an ATM" endures as an interview question because it's one of the few LLD prompts where the *money* itself, not just the software, is on the line if a design is subtly wrong — a network timeout that occurs a half-second after cash physically leaves the tray is not a hypothetical edge case, it's a routine occurrence any real deployment must survive without either giving away free money or silently debiting a customer for cash they never received. It rewards a candidate who reaches past "just call the bank API and dispense the cash" and recognizes that the *order* and *atomicity* of "reserve," "dispense," and "confirm" is the entire question. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Sealed interfaces model session state and transaction types precisely; records keep cassette/reservation snapshots immutable |
| Runnable artifact | HTML5 Canvas + vanilla JavaScript | Single-file, dependency-free, lets the reserve-then-confirm fix (and its failure modes) be triggered and watched live |
| Cash dispensing | Bounded-knapsack-style dynamic programming | Ordinary change-making assumes unlimited supply; a real cassette's finite note count needs a genuinely different, constrained formulation |
| Withdrawal correctness | Reserve-then-confirm, with a durable reconciliation log | The only pattern that survives a network failure at *any* point without either overpaying or losing a debit |
| Balance concurrency | Database-level atomic debit (optimistic or pessimistic, per this series' own BookMyShow guide) | The same double-booking-shaped race this series has solved for seats and inventory, applied here to an account balance |

---

# 5. Project Structure

```text
atm-system/
├── src/main/java/com/example/atm/
│   ├── domain/
│   │   └── Card.java, Account.java, Cassette.java, Transaction.java   // §10
│   ├── session/
│   │   ├── SessionState.java (State pattern)                          // §12-13
│   │   └── PinVerifier.java (retry-limit lockout)                     // §15-16
│   ├── dispensing/
│   │   ├── ConstrainedChangeMaker.java (bounded DP)                    // §21
│   │   └── UndispensableAmountException.java                          // §23
│   ├── withdrawal/
│   │   ├── ReserveThenConfirmWorkflow.java                            // §26-27
│   │   └── ReconciliationJob.java                                     // §29
│   ├── balance/
│   │   └── AccountLedger.java (atomic debit)                          // §32
│   └── transaction/
│       └── TransactionStrategy.java (Strategy interface)               // §37
├── src/test/java/com/example/atm/
│   ├── ConstrainedDispenseTest.java
│   ├── ReconciliationAmbiguousOutcomeTest.java
│   └── ConcurrentWithdrawalTest.java
└── artifact/
    └── atm-arena.html   -- the runnable simulator, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design an ATM."*

Even for a machine everyone has personally used, the intentionally open prompt needs narrowing: is the interesting part the physical mechanics (card reader, cash tray), or the correctness of the money movement against a bank's own ledger — for this guide, and for most interviewers, it's the latter. Does the design need to account for network unreliability between the ATM and the bank's core system, or can that be assumed away? Are multiple transaction types in scope (withdrawal, balance inquiry, deposit, transfer), or is withdrawal the whole story? Does the design need to survive two withdrawal attempts on the same account at nearly the same instant, from two different machines? The answers reshape which parts of the design carry real weight, and this guide's own focus — the reserve-then-confirm withdrawal flow — is precisely the part worth spending the most interview time on.

---

# 7. Functional Requirements

- **Authenticate a card and PIN**, locking the card after a configured number of consecutive failed attempts.
- **Dispense cash** for a withdrawal, using the fewest possible notes from the machine's actual, currently-available cassette inventory.
- **Debit the customer's account** exactly once per successful withdrawal, even across a network failure at any point in the transaction.
- **Reconcile ambiguous outcomes**: if the system cannot immediately determine whether a debit succeeded after cash was dispensed, resolve it automatically and correctly once connectivity returns.
- **Prevent overdraft from concurrent withdrawals**: two near-simultaneous withdrawal attempts on the same account must never both succeed past the available balance.
- **Support balance inquiries, deposits, and transfers** alongside withdrawals, through the same session and authentication flow.
- **Degrade safely** when network connectivity is lost mid-session, rather than behaving unpredictably.

---

# 8. Non-Functional Requirements

- **No free money, ever**: cash must never be dispensed without a corresponding, durable debit eventually taking hold — this outranks every other consideration in this guide.
- **No lost debits**: a customer must never be charged for cash they did not actually receive.
- **Bounded reconciliation latency**: an ambiguous transaction must resolve to a definite, correct outcome within a short, predictable window once connectivity is restored, not remain unresolved indefinitely.
- **Correctness under concurrency**: a balance check-then-debit must be atomic, regardless of how many machines or channels might be acting on the same account simultaneously.
- **Fail-safe degradation**: when in doubt (network lost, cassette state uncertain), the machine must default to the choice that protects the bank and the customer from an incorrect financial outcome, even at the cost of declining a transaction that might have been safe to complete.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Card** — the customer's identifier at the machine, paired with a PIN for authentication.
- **Account** — the customer's balance at the bank, the actual source of truth this entire system exists to protect.
- **Cassette** — a physical compartment holding a countable, finite number of notes of one specific denomination.
- **Session** — one customer's interaction with the machine from card insertion to card ejection, with an explicit lifecycle.
- **Transaction** — a specific requested operation (withdrawal, deposit, inquiry, transfer) within a session.
- **Reservation** — a temporary, provisional hold against an account's balance, made *before* cash physically dispenses, and only converted into a final, durable debit once dispensing is confirmed to have actually happened.

---

# 10. Identifying the Core Domain Entities

```java
public record Card(String cardId, String accountId) { }

public class Account {
    private final String accountId;
    private long balanceCents;
    private long reservedCents; // held against pending withdrawals, NOT yet a final debit -- §26-27
    public long availableCents() { return balanceCents - reservedCents; }
}

public class Cassette {
    private final int denominationCents;
    private int noteCount;
    public boolean hasAtLeast(int notes) { return noteCount >= notes; }
}

public record Transaction(String transactionId, String accountId, TransactionType type, long amountCents, Instant createdAt) { }
public enum TransactionType { WITHDRAWAL, DEPOSIT, INQUIRY, TRANSFER }
```

Splitting `Account` into `balanceCents` (the durable, confirmed truth) and `reservedCents` (a provisional hold not yet finalized) is the single modeling decision this entire guide's correctness story depends on — `availableCents()` is what every other withdrawal attempt actually checks against, which is precisely what makes a reservation visible to concurrent requests *before* it's ever converted into a final debit.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Card inserted ----->|       SessionState        |
                    |     (State pattern)       |
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    |      PinVerifier            |------>|  Card lockout, §15-16   |
                    +------------------------+       +------------------------+
                                |
                    +-----------v--------------+
                    |    TransactionStrategy      |
                    |  Withdrawal | Deposit |      |
                    |  Inquiry | Transfer          |
                    +-----------+--------------+
                                |
                    +-----------v--------------+       +------------------------+
                    | ReserveThenConfirmWorkflow |------>|   AccountLedger          |
                    +-----------+--------------+       |   (atomic debit, §32)   |
                                |
                    +-----------v--------------+       +------------------------+
                    | ConstrainedChangeMaker      |------>|   ReconciliationJob      |
                    | (bounded cassette DP)       |       |   (background, §29)      |
                    +------------------------+       +------------------------+
```

Every mechanism this guide builds sits along one linear path — authenticate, select transaction, reserve, dispense, confirm — with reconciliation acting as a safety net specifically for the case where "confirm" itself never definitively resolves.

---

# 12. Follow-up Question 2 — "Model the ATM Session as a State Machine — Why Not Just a Few Booleans?"

> **Interviewer:** *"`cardInserted`, `pinVerified`, `transactionInProgress` — three flags. What's wrong with tracking a session's status that way?"*

Exactly the same problem this series has diagnosed for a parking spot, an elevator car, a booked seat, and a vending machine — independent booleans can be set into combinations that are physically nonsensical (`transactionInProgress = true` while `pinVerified = false` describes a transaction proceeding without ever having authenticated the customer, which should be structurally impossible, not merely "shouldn't happen if the code is careful"). The fix is the same explicit State pattern this series has applied consistently.

---

# 13. The ATM Session as a State Machine

```text
IDLE ----insertCard()----> AWAITING_PIN
AWAITING_PIN ----correctPin()----> SELECTING_TRANSACTION
AWAITING_PIN ----incorrectPin(), attempts remaining----> AWAITING_PIN (unchanged, attempt counted)
AWAITING_PIN ----incorrectPin(), retry limit reached----> CARD_RETAINED
SELECTING_TRANSACTION ----chooseTransaction()----> PROCESSING
PROCESSING ----complete()----> IDLE (card ejected)
ANY STATE ----networkLost()----> DEGRADED (§34-35)

Illegal transitions (structurally PREVENTED, not just discouraged by convention):
  IDLE.correctPin()              -- no card inserted yet, nothing to authenticate against
  SELECTING_TRANSACTION.insertCard() -- a session is already active, a second card mid-session is nonsensical
  CARD_RETAINED.chooseTransaction()  -- the card is gone, no further transactions are possible this session
```

This is the same discipline this series has used for every prior guide's core lifecycle — a small, explicit set of phases, each exposing only the operations legal from it, with the state object itself deciding what happens next rather than scattered `if` checks.

---

# 14. Implementing the State Pattern for an ATM Session

```java
public interface SessionState {
    SessionState insertCard(AtmSession session, Card card);
    SessionState submitPin(AtmSession session, String pin);
    SessionState chooseTransaction(AtmSession session, TransactionType type);
}

public class AwaitingPinState implements SessionState {
    private int attemptsRemaining;

    @Override
    public SessionState submitPin(AtmSession session, String pin) {
        if (pinVerifier.matches(session.card(), pin)) {
            return new SelectingTransactionState();
        }
        attemptsRemaining--;
        if (attemptsRemaining <= 0) {
            session.retainCard();
            return new CardRetainedState();
        }
        return this; // stay here, one fewer attempt remaining
    }
    @Override
    public SessionState insertCard(AtmSession session, Card card) {
        throw new IllegalStateException("A card is already inserted");
    }
    @Override
    public SessionState chooseTransaction(AtmSession session, TransactionType type) {
        throw new IllegalStateException("PIN not yet verified");
    }
}
```

`attemptsRemaining` living *inside* the state object itself (not as a separate field on `AtmSession`) is a small but deliberate choice — it keeps the retry count scoped exactly to the phase it's meaningful in, and guarantees a fresh session (a new card insertion) can never accidentally inherit a stale attempt count from a previous, unrelated session.

---

# 15. Follow-up Question 3 — "Three Wrong PINs — How Do You Handle Lockout and Card Retention Correctly?"

> **Interviewer:** *"A customer enters the wrong PIN three times. What exactly happens, and why does it matter that the retry count lives where you just put it?"*

The retry limit must be enforced **per card**, durably, not merely per session in local machine memory — otherwise a customer (or an attacker) could simply eject the card and reinsert it to reset their attempt count, defeating the entire lockout mechanism. The count needs to be checked against (and incremented in) the bank's own durable record for that card, with the *local* session state (§14) reflecting, not owning, that durable count.

---

# 16. PIN Verification and Retry-Limit Lockout

```java
public class PinVerifier {
    private static final int MAX_ATTEMPTS = 3;

    public PinCheckResult verify(Card card, String enteredPin) {
        CardRecord record = cardRepository.findById(card.cardId()); // the DURABLE source of truth
        if (record.isLocked()) {
            return PinCheckResult.cardLocked();
        }
        if (hashPin(enteredPin).equals(record.pinHash())) {
            cardRepository.resetFailedAttempts(card.cardId()); // a SUCCESSFUL entry clears the counter
            return PinCheckResult.verified();
        }
        int updatedAttempts = cardRepository.incrementFailedAttempts(card.cardId());
        if (updatedAttempts >= MAX_ATTEMPTS) {
            cardRepository.lockCard(card.cardId());
            return PinCheckResult.lockedJustNow();
        }
        return PinCheckResult.incorrect(MAX_ATTEMPTS - updatedAttempts);
    }
}
```

Storing only a **hash** of the PIN, never the PIN itself, is non-negotiable even in a design-focused guide — comparing `hashPin(enteredPin)` against a stored hash means the actual PIN is never persisted anywhere in a form that could be read back if the storage were ever compromised.

---

# 17. Class Diagram: The ATM Session Core

```text
+------------------------+        +------------------------+
|       AtmSession           |------->|      SessionState         |
|  insertCard(), submitPin(),|       |    <<interface>>          |
|  chooseTransaction()       |       |  insertCard(), submitPin()|
+-----------+--------------+        +-----------+--------------+
            |                              ^      ^      ^      ^
            v                        +----------+ +--------+ +--------+ +----------------+
+------------------------+          | Idle     | | Awaiting| |Selecting| | CardRetained  |
|      PinVerifier           |          | State    | | Pin     | |Transact.| | State          |
+------------------------+          +----------+ | State   | | State  | +----------------+
                                                     +--------+ +--------+
```

Every state transition method returns the *next* state object rather than mutating a bare field in place, exactly the pattern this series has used for every prior state machine — which is what makes each transition's legality visible in which interface method a given state class chooses to implement meaningfully versus throw from.

---

# 18. Follow-up Question 4 — "How Do You Decide Which Notes to Dispense, Given Limited Cassette Inventory Per Denomination?"

> **Interviewer:** *"A customer requests $140. The machine has $100, $50, and $20 notes. Design the algorithm that decides exactly which notes to dispense — and account for the fact that each cassette holds a finite, countable number of notes, not an unlimited supply."*

This is the same *shape* of problem this series' own Vending Machine guide solved with change-making — minimize the number of notes/coins used — but with one critical difference that changes the algorithm entirely: a cassette's note count is **finite and known**, and the optimal answer must respect it. A combination that would be optimal against an unlimited supply (say, four $100 notes for $400) is simply unusable if the $100 cassette currently holds only two.

---

# 19. The Bounded-Supply Cash Dispensing Problem

```text
Vending Machine's change-making (this series' own prior guide): unlimited supply of each
  denomination assumed -- the only question is WHICH denominations to use, in what quantity,
  to minimize total count.

ATM cash dispensing: EACH denomination has a specific, currently-available note count --
  the algorithm must never propose using more notes of a denomination than the cassette
  actually holds, even if doing so would otherwise be the mathematically fewest-notes answer.

Customer requests $140. Cassettes: $100 x2, $50 x1, $20 x10.
  Unconstrained optimal:  100 + 20 + 20 = 3 notes (if $100 supply were unlimited... but it's fine here, 2 are enough)
  Actually available:     100 + 20 + 20 = 3 notes -- USES only 1 of the 2 available $100 notes, fine
  BUT if the request were $340 with only 2x$100 available: 100+100+100+20+20 is IMPOSSIBLE
    (needs three $100 notes, only two exist) -- the algorithm must find the best combination
    that respects EVERY cassette's actual remaining count, not just the theoretical minimum.
```

This is precisely the classic **bounded knapsack** problem, not the simpler unbounded change-making problem this series' Vending Machine guide solved — each denomination now has a maximum usable quantity, and the dynamic programming formulation must track that bound explicitly rather than assuming any denomination is always available in whatever quantity the math happens to want.

---

# 20. Why This Differs from Vending Machine's Change-Making

```text
Unbounded (Vending Machine, this series' own prior guide):
  minCoins[amount] = min over every denomination d <= amount of (1 + minCoins[amount - d])
  -- ANY number of a given denomination can be used, without limit

Bounded (ATM, this guide):
  minNotes[amount][d] = min over 0..maxAvailable[d] copies of denomination d used,
                        PLUS the best answer for the remaining amount using ONLY
                        denominations processed so far
  -- this is a genuinely two-dimensional recurrence (amount AND which denominations/how
     many of each have already been committed), not the simpler one-dimensional recurrence
     unbounded change-making allows
```

Recognizing that "the same algorithm, just add a supply check" is *not* actually sufficient here is the key insight this follow-up is testing — a bounded supply changes the recurrence's shape, not merely adds a guard clause on top of the unbounded one.

---

# 21. Implementing Denomination-Constrained Dispensing

```java
public class ConstrainedChangeMaker {
    public Map<Integer, Integer> dispense(int amountCents, List<Cassette> cassettes) {
        // dp[a] = minimum notes to make amount 'a', considering cassettes processed SO FAR
        int[] dp = new int[amountCents + 1];
        Arrays.fill(dp, Integer.MAX_VALUE);
        dp[0] = 0;
        // track how many of EACH denomination were used to reach each achievable amount
        Map<Integer, int[]> usedCountByDenomination = new HashMap<>();

        for (Cassette cassette : cassettes) {
            int denom = cassette.denominationCents();
            int maxAvailable = cassette.noteCount();
            int[] usedThisDenom = new int[amountCents + 1];
            int[] nextDp = dp.clone();

            for (int a = denom; a <= amountCents; a++) {
                for (int count = 1; count <= maxAvailable && count * denom <= a; count++) {
                    if (dp[a - count * denom] != Integer.MAX_VALUE
                            && dp[a - count * denom] + count < nextDp[a]) {
                        nextDp[a] = dp[a - count * denom] + count;
                        usedThisDenom[a] = count;
                    }
                }
            }
            dp = nextDp;
            usedCountByDenomination.put(denom, usedThisDenom);
        }

        if (dp[amountCents] == Integer.MAX_VALUE) {
            throw new UndispensableAmountException(amountCents); // §22-23
        }
        return reconstructNoteBreakdown(amountCents, cassettes, usedCountByDenomination);
    }
}
```

Processing cassettes **one denomination at a time**, each time deciding how many of *that specific* denomination to commit to before moving to the next, is what correctly encodes the bounded-supply constraint into the recurrence itself — every `dp[a]` value at any point already reflects only cassettes considered so far, never assuming availability from a denomination not yet accounted for.

---

# 22. Follow-up Question 5 — "What If the Requested Amount Can't Be Made Exactly, Even Though Total Cash Is Sufficient?"

> **Interviewer:** *"The machine has plenty of total cash, but the customer's requested amount genuinely cannot be assembled from the specific notes remaining. What does the machine do?"*

The machine must **detect this before ever attempting to physically dispense anything**, and offer the customer a clear, actionable alternative — either a nearby amount that *can* be exactly assembled, or a straightforward decline with an explanation — rather than either failing silently or dispensing an incorrect amount because the algorithm quietly rounded or substituted denominations it wasn't actually asked to use.

---

# 23. Detecting and Handling Undispensable Amounts

```java
public class UndispensableAmountException extends RuntimeException {
    private final int requestedCents;
    public UndispensableAmountException(int requestedCents) {
        super("Cannot dispense $" + (requestedCents / 100.0) + " with current cassette inventory");
        this.requestedCents = requestedCents;
    }
}

// caller (the withdrawal transaction handler):
try {
    Map<Integer, Integer> notes = changeMaker.dispense(requestedCents, currentCassettes);
    // proceed to reserve-then-confirm, §26-27 -- dispensing is now KNOWN to be physically possible
} catch (UndispensableAmountException e) {
    int nearestDispensable = findNearestDispensableAmount(requestedCents, currentCassettes);
    return WithdrawalResult.suggestAlternative(nearestDispensable);
}
```

Running the dispensability check **before** ever touching the reserve-then-confirm workflow (§26-27) is deliberate ordering — there is no reason to place a provisional hold against a customer's account for an amount the machine has already determined it cannot physically produce.

---

# 24. Follow-up Question 6 — "The Cash Dispenses, But the Network Call Confirming the Debit Times Out. What Happens to the Customer's Balance?"

> **Interviewer:** *"Walk through this precisely: the machine physically dispenses $200, and at that exact moment, the network connection to the bank's core system drops before the debit confirmation returns. Does the customer keep the $200 for free, or get charged twice if they call to complain and a retry happens?"*

This is the single most important question in the entire ATM design space, and a naive implementation gets it wrong by ordering things backwards: if the machine **debits first, then dispenses**, a crash or timeout *after* the debit but *before* dispensing charges a customer for cash they never received. If it **dispenses first, then debits**, a timeout *after* dispensing but *before* the debit confirms risks the debit never happening at all — free money. Neither naive ordering is safe; the fix requires a third, distinct step in between.

---

# 25. The Cash-Dispensed-but-Debit-Unconfirmed Problem

```text
NAIVE (debit first, then dispense):
  1. Debit account -- SUCCEEDS, durably
  2. Dispense cash -- machine JAMS or crashes here
  RESULT: customer charged, received NOTHING -- a lost debit, a real customer harm

NAIVE (dispense first, then debit):
  1. Dispense cash -- SUCCEEDS, cash physically leaves the tray
  2. Debit account -- NETWORK TIMES OUT here, before confirmation returns
  RESULT: customer received cash, account was NEVER debited -- free money, a real bank loss

Both naive orderings have a WINDOW where the physical event (cash leaving) and the financial
event (the debit taking hold) can become permanently inconsistent with each other.
```

The fix isn't a different ordering of the same two steps — it's recognizing that a *third* state needs to exist in between: a **provisional reservation**, held durably *before* cash ever dispenses, that gets **confirmed** into a final debit only *after* dispensing is known to have actually happened, and can be safely **rolled back** if dispensing never occurs at all.

---

# 26. Designing the Fix: Reserve-Then-Confirm, Not Debit-Then-Dispense

```text
1. RESERVE: place a durable hold against the account (balanceCents unchanged, reservedCents
   increases) -- this is a REAL database write, survives a crash, and makes the reserved
   amount invisible to OTHER concurrent withdrawal attempts (§30-32) immediately
2. DISPENSE: the machine physically releases the cash -- a MECHANICAL action, confirmed by
   a sensor, independent of the network entirely
3. CONFIRM: ONLY AFTER dispensing is sensor-confirmed, convert the reservation into a final
   debit -- balanceCents decreases, reservedCents decreases back to reflect the hold is gone

If dispensing FAILS (jam, sensor doesn't confirm release): the reservation is released
  WITHOUT ever becoming a debit -- the customer's balance is completely unaffected,
  exactly as if the transaction never happened.

If confirmation ITSELF fails (network drops after dispensing but before the confirm write
  lands): the reservation is left in a RECOVERABLE, ambiguous state -- resolved by
  reconciliation (§28-29), NEVER left permanently unresolved.
```

The reservation is what makes this design correct under *every* failure point — a crash before dispensing simply means no cash ever left and the reservation rolls back cleanly; a crash after dispensing but before confirmation leaves a durable, recoverable record that dispensing already happened, which reconciliation can act on with certainty rather than having to guess.

---

# 27. Implementing a Two-Phase Withdrawal: Reserve, Dispense, Confirm

```java
public class ReserveThenConfirmWorkflow {
    public WithdrawalResult withdraw(String accountId, int amountCents, List<Cassette> cassettes) {
        String reservationId = accountLedger.reserve(accountId, amountCents); // durable write #1

        boolean dispensed;
        try {
            dispensed = cashDispenser.dispense(amountCents, cassettes); // mechanical, sensor-confirmed
        } catch (Exception mechanicalFailure) {
            accountLedger.releaseReservation(reservationId); // no cash left -- roll back cleanly
            return WithdrawalResult.failed("Dispensing error -- your account was not charged");
        }

        if (!dispensed) {
            accountLedger.releaseReservation(reservationId);
            return WithdrawalResult.failed("Unable to dispense -- your account was not charged");
        }

        try {
            accountLedger.confirmReservation(reservationId); // durable write #2 -- NOW it's a final debit
            return WithdrawalResult.success(amountCents);
        } catch (NetworkException confirmationFailed) {
            // cash IS ALREADY OUT. The reservation record durably shows this. Reconciliation
            // (§28-29) will resolve this to a confirmed debit once connectivity returns --
            // this is NOT an error state requiring a retry from here; it's a HANDOFF.
            return WithdrawalResult.pendingReconciliation(reservationId);
        }
    }
}
```

The `catch (NetworkException confirmationFailed)` branch deliberately does **not** attempt to retry the confirmation inline, and does **not** report a generic failure to the customer — it hands the now-ambiguous-but-recoverable reservation off to a background process that has the durable record it needs to resolve this correctly, exactly once, whenever connectivity returns.

---

# 28. Follow-up Question 7 — "The Confirm Call Also Fails or Times Out After Dispensing. How Do You Reconcile Eventually?"

> **Interviewer:** *"You've handed the ambiguous case off to reconciliation. What does that background process actually do, concretely, to resolve it correctly?"*

The reservation record itself already durably captures everything needed: which account, how much, and — critically — whether the *mechanical* dispense sensor confirmed cash actually left the tray, recorded locally on the ATM regardless of network status. Reconciliation's job is simply to read that local, already-known fact and apply it: if dispensing was sensor-confirmed, finalize the debit; if it wasn't, release the reservation — the ambiguity was only ever about the *network call*, never about the underlying fact of what actually happened.

---

# 29. Reconciliation: A Background Job That Resolves Ambiguous Transactions

```java
public class ReconciliationJob {
    @Scheduled(fixedRate = 30_000) // runs regularly, independent of any single transaction's own retry
    public void resolveAmbiguousReservations() {
        List<Reservation> pending = reservationRepository.findPendingReconciliation();
        for (Reservation reservation : pending) {
            if (reservation.wasDispenseSensorConfirmed()) { // read from the ATM's OWN local, durable log
                accountLedger.confirmReservation(reservation.id()); // finalize the debit, exactly once
            } else {
                accountLedger.releaseReservation(reservation.id()); // no cash left -- release cleanly
            }
            reservation.markReconciled();
        }
    }
}
```

Running this job on a fixed schedule, independent of any individual transaction's own request lifecycle, is what guarantees an ambiguous reservation never remains unresolved indefinitely — the customer's own session may have already ended, the ATM may have moved on to serving other customers, but this background process keeps checking until every pending reservation reaches a final, correct state.

---

# 30. Follow-up Question 8 — "Two ATMs Try to Withdraw from the Same Account at Nearly the Same Instant. How Do You Prevent Overdraft?"

> **Interviewer:** *"A customer has $100. They insert a card at one ATM and, at nearly the same instant, their mobile banking app initiates a $100 withdrawal too. Both check the balance, both see $100 available, both attempt to proceed. What stops both from succeeding?"*

This is the exact same check-then-act race this series has diagnosed for a parking spot, a booked seat, and vending-machine inventory — reading `availableCents()` and only afterward reserving against it has a window where a second reader can observe the same pre-reservation balance before the first reservation actually lands. The fix is identical in shape: collapse the check and the reservation into a single atomic operation.

---

# 31. The Double-Withdrawal Race on a Shared Balance

```text
Time    ATM A (Channel 1)                          Mobile App (Channel 2)
t0      reads availableCents() = $100 (balance $100, reserved $0)
t1                                                  reads availableCents() = $100  -- SAME stale read
t2      reserve($100) -- reservedCents now $100
t3                                                  reserve($100) -- reservedCents now $200
                                                     -- BOTH channels believed $100 was available
                                                        and BOTH proceeded -- a genuine overdraft risk
```

Both channels genuinely saw $100 available at the moment they checked — the bug is that nothing prevented a second read from happening before the first reservation had a chance to invalidate it, precisely the same shape this series has closed with either pessimistic or optimistic locking every time it has appeared.

---

# 32. Implementing Concurrency-Safe Balance Debit

```java
public class AccountLedger {
    @Transactional
    public String reserve(String accountId, int amountCents) {
        // SELECT ... FOR UPDATE -- exactly this series' own BookMyShow guide's pessimistic
        // locking approach (§16-18 there), applied here to an account row instead of a seat
        Account account = accountRepository.findByIdForUpdate(accountId);
        if (account.availableCents() < amountCents) {
            throw new InsufficientFundsException(accountId);
        }
        account.setReservedCents(account.reservedCents() + amountCents);
        accountRepository.save(account);
        return reservationRepository.create(accountId, amountCents);
        // the row lock is held until this transaction commits -- the SECOND channel's own
        // SELECT ... FOR UPDATE simply WAITS, then re-reads the NOW-CORRECT availableCents(),
        // and correctly sees insufficient funds if the first reservation already consumed them
    }
}
```

Using `SELECT ... FOR UPDATE` (rather than optimistic locking) is a deliberate choice specifically for this operation — an account balance check is short, fast, and genuinely safety-critical enough that briefly serializing concurrent access to the *same* account is a small, acceptable cost, exactly the tradeoff this series' own BookMyShow guide worked through in detail for choosing between the two families.

---

# 33. Class Diagram: The Withdrawal and Reconciliation Flow

```text
+------------------------+        +------------------------+
| ReserveThenConfirmWorkflow |------->|     AccountLedger        |
|  withdraw()                |       |  reserve(), confirm(),   |
+-----------+--------------+       |  releaseReservation()    |
            |                       +-----------+--------------+
            v                                    |
+------------------------+                       v
| ConstrainedChangeMaker    |          +------------------------+
+------------------------+          |       Account              |
                                       |  balanceCents,            |
                                       |  reservedCents            |
                                       +------------------------+
                                                  ^
                                                  |
                                       +------------------------+
                                       |    ReconciliationJob      |
                                       |    (background, §29)      |
                                       +------------------------+
```

`Account`'s own `balanceCents`/`reservedCents` split (§10) is the shared state every other component in this diagram ultimately reads and writes — the reservation workflow, the reconciliation job, and the concurrency-safe lock all exist specifically to protect the integrity of these two numbers.

---

# 34. Follow-up Question 9 — "What Happens If the ATM Loses Network Connectivity Mid-Transaction?"

> **Interviewer:** *"The network drops entirely, not just one call timing out — the machine has no way to reach the bank at all for the next several minutes. What should it do?"*

The machine must **fail safe**: refuse to start any new transaction that requires reserving funds (since it cannot durably record a reservation the bank's own ledger would recognize), while still allowing whatever it can safely do purely locally — displaying a clear "temporarily unable to process transactions" message, ejecting any inserted card, and logging the outage locally for later audit, rather than guessing at an offline balance or allowing a transaction it can't guarantee will reconcile correctly.

---

# 35. Offline/Degraded Mode and Fail-Safe Defaults

```java
public class DegradedModeState implements SessionState {
    @Override
    public SessionState chooseTransaction(AtmSession session, TransactionType type) {
        display.show("Temporarily unable to process transactions. Please try again shortly.");
        session.ejectCard();
        return new IdleState();
    }
    // insertCard() and submitPin() similarly refuse to proceed while degraded
}
```

Refusing new transactions entirely during a genuine network outage — rather than offering a limited "offline withdrawal allowance" some real deployments do implement — is the more conservative, guide-appropriate default: any offline allowance reintroduces exactly the reconciliation risk this guide's §24-29 spent so much effort closing, and should only ever be adopted as a deliberate, explicit business tradeoff, not a default correctness posture.

---

# 36. Follow-up Question 10 — "How Do You Support Multiple Transaction Types — Withdrawal, Balance Inquiry, Deposit, Transfer — Cleanly?"

> **Interviewer:** *"Withdrawal has this whole reserve-then-confirm dance. A balance inquiry needs none of that. How do you support both without one polluting the other's logic?"*

By keeping `SessionState.chooseTransaction` (§14) delegate to a pluggable `TransactionStrategy` per transaction type, rather than branching internally — a balance inquiry's strategy is trivially simple (one read, no reservation, no dispensing at all), and a withdrawal's strategy is the full reserve-then-confirm workflow this guide has built, with neither one needing to know the other exists.

---

# 37. Pluggable Transaction Types

```java
public interface TransactionStrategy {
    TransactionResult execute(AtmSession session, TransactionRequest request);
}
```

Every transaction type — however simple or however involved — implements this identical interface, exactly the Strategy pattern this series has used for AI difficulty, dispatch algorithms, and pricing schemes throughout: adding a new transaction type is a new class, never a change to `AtmSession`'s own session logic.

---

# 38. Implementing Transaction Strategies

```java
public class InquiryStrategy implements TransactionStrategy {
    @Override
    public TransactionResult execute(AtmSession session, TransactionRequest request) {
        Account account = accountRepository.findById(session.card().accountId());
        return TransactionResult.balance(account.availableCents()); // no reservation needed at all
    }
}

public class WithdrawalStrategy implements TransactionStrategy {
    private final ReserveThenConfirmWorkflow workflow; // the full flow from §26-27

    @Override
    public TransactionResult execute(AtmSession session, TransactionRequest request) {
        return workflow.withdraw(session.card().accountId(), request.amountCents(), session.currentCassettes());
    }
}
```

`InquiryStrategy`'s entire implementation is a single read — it has no access to, and no need for, any of the reservation/dispensing machinery `WithdrawalStrategy` depends on, which is precisely the payoff of keeping transaction types behind a narrow, uniform interface rather than one large class handling every case internally.

---

# 39. Follow-up Question 11 — "How Do You Support Cash/Check Deposits, and Reconcile a Deposited Amount That Hasn't Been Human-Verified Yet?"

> **Interviewer:** *"A customer deposits an envelope claiming it contains $500. The machine can't verify that claim immediately — a human has to open and count it later. Does the account get credited $500 right away?"*

Crediting the claimed amount **provisionally**, clearly marked as pending verification, mirrors the exact same reserve-then-confirm shape this guide already built for withdrawals — just running in the opposite financial direction. The customer sees a provisional credit immediately (a reasonable UX expectation), but the bank's own ledger keeps it distinguishable from a fully confirmed balance until a human verification step either confirms the claimed amount or adjusts it.

---

# 40. Deposit Handling and Provisional Crediting

```java
public class DepositStrategy implements TransactionStrategy {
    @Override
    public TransactionResult execute(AtmSession session, TransactionRequest request) {
        String provisionalCreditId = accountLedger.creditProvisionally(
            session.card().accountId(), request.claimedAmountCents());
        // the customer sees this reflected in their balance display immediately, but it
        // remains PROVISIONAL until a human verification step confirms or adjusts it --
        // structurally identical to a withdrawal's reservation, just crediting instead of debiting
        return TransactionResult.provisionalDeposit(provisionalCreditId, request.claimedAmountCents());
    }
}
```

Deliberately reusing the *shape* of the reservation mechanism (a durable, provisional entry, later confirmed or adjusted by a separate, authoritative step) rather than inventing a new mechanism for deposits is what keeps this guide's core correctness pattern — never let an unconfirmed financial event masquerade as a final one — consistent across every transaction type, not just withdrawals.

---

# 41. Class Diagram: Full ATM System

```text
+------------------------+        +------------------------+
|       AtmSession           |------->|    TransactionStrategy    |
|   (State pattern, §13-14) |       |    <<interface>>          |
+------------------------+        +-----------+--------------+
                                                  ^      ^      ^      ^
                                    +----------+ +--------+ +--------+ +----------------+
                                    | Inquiry  | | With-  | | Deposit| | Transfer       |
                                    | Strategy | | drawal | |Strategy| | Strategy       |
                                    +----------+ | Strategy|+--------+ +----------------+
                                                  +--------+
                                                       |
                                                       v
                                    +------------------------+        +------------------------+
                                    | ReserveThenConfirmWorkflow |------>|     AccountLedger        |
                                    +------------------------+        +------------------------+
                                                       |                            ^
                                                       v                            |
                                    +------------------------+        +------------------------+
                                    | ConstrainedChangeMaker    |        |    ReconciliationJob     |
                                    +------------------------+        +------------------------+
```

Tracing this diagram left to right is the entire system: session state gates what's legal, transaction strategies each handle their own kind of operation, and the ones that move real money (withdrawal, deposit) both funnel through the same reserve/confirm shape backed by one shared, concurrency-safe ledger.

---

# 42. Capacity Estimation: Cassette Refill Cadence and Transaction Throughput

```text
Assume: a moderately busy machine serves 1 transaction every 90 seconds on average, with a
        typical withdrawal averaging $120 and cassettes holding 2,000 notes each at refill

Transactions/day ≈ (24 * 3600) / 90                    ≈ 960 transactions/day
Cash dispensed/day ≈ 960 * $120 (assume ~60% are withdrawals) ≈ $69,000/day

A $20 cassette starting at 2,000 notes, if $20 notes make up roughly a third of dispensed
  value, depletes in: (2,000 * $20) / ($69,000 * 0.33) ≈ 1.75 days -- confirming that DAILY
  or near-daily refill cadence is the realistic operational baseline this design must support,
  which is precisely why §18-23's bounded-supply dispensing (not an unlimited-supply
  assumption) is a genuine, load-bearing requirement, not a theoretical nicety.
```

This capacity estimate is the concrete justification for treating cassette depletion as a real, routine operating condition rather than a rare edge case — at this refill cadence, a real deployment hits the "requested amount temporarily undispensable" scenario (§22-23) often enough that handling it gracefully is a core requirement, not an afterthought.

---

# 43. Full Worked Example: One Withdrawal, Traced End to End

```text
1. Customer inserts card, enters correct PIN on the first attempt: AwaitingPinState.submitPin()
   -> SelectingTransactionState (§13-14, §16)
2. Customer requests a $140 withdrawal: WithdrawalStrategy.execute() invoked (§37-38)
3. ConstrainedChangeMaker.dispense(14000, cassettes) -- bounded DP finds a valid combination
   using currently-available notes, e.g. one $100 + two $20 notes (§19-21)
     a. If NO valid combination existed: UndispensableAmountException caught, customer
        offered an alternative amount, NO reservation ever placed (§22-23)
4. ReserveThenConfirmWorkflow.withdraw() begins:
     a. AccountLedger.reserve() -- SELECT ... FOR UPDATE, durable hold placed, $140 now
        invisible to any OTHER concurrent withdrawal attempt on this account (§30-32)
     b. CashDispenser.dispense() -- mechanical action, sensor confirms release
     c. AccountLedger.confirmReservation() -- SUCCEEDS this time, reservation becomes a
        final debit (§26-27)
5. Customer receives $140 and a receipt; session returns to IdleState, card ejected

--- A different customer, same machine, network drops mid-confirmation ---

6. Steps 1-4b proceed identically, but 4c's confirmReservation() call TIMES OUT
7. WithdrawalResult.pendingReconciliation() returned -- session ends normally from the
   customer's point of view (cash was already dispensed, receipt printed)
8. ReconciliationJob runs on its next scheduled pass, reads the LOCAL sensor-confirmed
   dispense record, and finalizes the debit -- exactly once, correctly, without the customer
   or a technician needing to do anything (§28-29)
```

Every mechanism this guide introduced via a follow-up question appears somewhere in these two traces — bounded-supply dispensing, reserve-then-confirm, concurrency-safe reservation, and background reconciliation are not independent, optional features, they are the actual steps every real withdrawal (successful or network-interrupted) passes through in this design.

---

# 44. Final Architecture Diagram

```text
Card + PIN --> [ SessionState ] --> [ PinVerifier: retry-limit lockout, durable per card ]
                       |
                       v
              [ TransactionStrategy: Inquiry | Withdrawal | Deposit | Transfer ]
                       |
                       v (for money-moving transactions)
        +------------------------+       +------------------------+
        | ConstrainedChangeMaker    |       |   AccountLedger          |
        | (bounded DP, §19-21)      |       |   reserve/confirm/       |
        +------------------------+       |   release, atomic (§32)  |
                                            +-----------+--------------+
                                                          |
                                            +-------------v-------------+
                                            |     ReconciliationJob        |
                                            |    (background, §28-29)      |
                                            +------------------------+
```

---

# 45. Design Patterns Used Throughout This Guide

- **State** — `SessionState` (§12-14) encodes exactly which operations are legal from the session's current phase, structurally preventing illegal operations (authenticating without a card, transacting without PIN verification) rather than relying on scattered runtime checks.
- **Strategy** — `TransactionStrategy` (§36-38) lets each transaction type vary independently of `AtmSession`'s own orchestration logic.
- **Facade** — `ReserveThenConfirmWorkflow` (§27) hides reservation, dispensing, and confirmation behind a single `withdraw()` entry point every money-moving transaction type calls.
- **Template Method** (implicit) — reserve, dispense, confirm/reconcile is a fixed skeleton every money-moving transaction (withdrawal, deposit) follows, varying only in direction and which specific mechanical action occupies the middle step.

---

# 46. SOLID Principles Applied

- **Single Responsibility** — `PinVerifier` only authenticates; `ConstrainedChangeMaker` only decides which notes to dispense; `AccountLedger` only manages reservations and debits — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new transaction type or a new dispensing algorithm each means implementing one interface, never modifying `AtmSession`'s own state transitions or the reserve-then-confirm workflow.
- **Liskov Substitution** — every `TransactionStrategy` implementation must honestly return a `TransactionResult` for any valid request, so `AtmSession` can invoke any of them uniformly without special-casing a particular type.
- **Interface Segregation** — `TransactionStrategy` exposes exactly one method, so a trivial `InquiryStrategy` isn't forced to depend on reservation or dispensing concepts only money-moving strategies actually need.
- **Dependency Inversion** — `ReserveThenConfirmWorkflow` depends on `AccountLedger` and `ConstrainedChangeMaker` as abstractions, never on concrete database or cassette-hardware implementations directly, so either can be swapped without touching the workflow itself.

---

# 47. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
A few independent boolean flags for session status         Explicit State pattern with legal transitions (§12-14)
Enforcing PIN retry limits only in local session memory      Durable, per-card retry count (§15-16)
Assuming unlimited note supply, like ordinary change-making   Bounded-supply constrained dispensing (§18-21)
Debiting the account BEFORE dispensing cash                  Reserve-then-confirm, never debit-first (§24-27)
Retrying a failed confirmation call inline, indefinitely       Hand off to background reconciliation (§28-29)
Checking balance, then reserving, as two separate steps        Atomic, lock-based reservation (§30-32)
Guessing an offline balance during a network outage            Fail safe: refuse new transactions (§34-35)
```

---

# 48. Testing Strategy

- **State transition tests** — assert every illegal transition (authenticating without a card, transacting before PIN verification) throws, and every legal transition produces the correct resulting state.
- **PIN lockout tests** — assert the retry count is enforced durably per card, surviving a card ejection and reinsertion within the same lockout window.
- **Bounded-dispense tests** — construct cassette configurations where an amount is dispensable, and others where it is genuinely not despite sufficient total cash, and assert the algorithm correctly distinguishes the two.
- **Reserve-then-confirm tests** — simulate a mechanical dispense failure (reservation must roll back cleanly) and a confirmation-network-timeout (reservation must be left in a reconcilable, not lost, state).
- **Reconciliation tests** — feed the reconciliation job a mix of sensor-confirmed and sensor-unconfirmed pending reservations, and assert each resolves to the correct final outcome exactly once.
- **Concurrent withdrawal tests** — fire many simultaneous withdrawal attempts against one account with a fixed balance and assert the total successfully reserved amount never exceeds what was actually available.

---

# 49. Suggested Future Enhancements

- **Contactless/mobile-initiated withdrawals** — a customer pre-authorizes an amount via a mobile app and completes dispensing at any ATM without inserting a physical card, reusing the exact same reserve-then-confirm workflow with a different session-initiation strategy.
- **Predictive cassette refill scheduling** — using historical withdrawal-amount distributions per machine to forecast which denominations will become undispensable soonest, informing refill routes proactively rather than reactively.
- **Multi-currency dispensing** — extending `ConstrainedChangeMaker` to select not just denominations but a currency, for machines serving international travelers, without changing the reserve-then-confirm workflow at all.
- **Biometric authentication** — an additional `SessionState` transition path alongside PIN verification, reusing the same retry-limit and lockout discipline already built for PINs.
- **Real-time fraud scoring** — layering anomaly detection (unusual withdrawal amounts, unusual times, unusual locations relative to the card's typical pattern) as an additional check before `AccountLedger.reserve()` ever proceeds, without touching the reservation mechanism's own correctness guarantees.

---

# 50. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Model the ATM session as an explicit state machine, including PIN retry-limit lockout, and explain why the retry count must live durably per card rather than in local session memory. (§12-16)
2. Design cash dispensing that minimizes note count, given a genuinely finite, countable supply per denomination — and explain precisely how this differs from ordinary unbounded change-making. (§18-21)
3. Design what happens when a requested amount cannot be exactly assembled from current cassette inventory, even though total cash is sufficient. (§22-23)
4. Cash dispenses, then the network call confirming the debit times out. Design the fix, precisely — don't just say "handle it," describe the actual states involved. (§24-27)
5. The confirmation call fails even after retrying. Design the background process that resolves this correctly, exactly once. (§28-29)
6. Two channels attempt to withdraw from the same account at nearly the same instant. Design the fix that prevents overdraft. (§30-32)
7. The ATM loses network connectivity entirely, mid-session. Design the fail-safe behavior, and justify why it's the correct default. (§34-35)
8. Extend the design to support deposits, reusing as much of the withdrawal machinery as genuinely applies. (§39-40)

---

# 51. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **a physical, irreversible action (cash leaving a tray) and a financial commitment (a debit taking hold) must never be allowed to become permanently inconsistent with each other, no matter where a failure occurs between them** — reserve-then-confirm exists specifically to insert a durable, recoverable checkpoint between those two events (§24-27); reconciliation exists because even that checkpoint's own confirmation can fail, and must still resolve correctly later (§28-29); bounded-supply dispensing exists because the physical world (a finite cassette) constrains what's honestly possible in a way an idealized unlimited-supply algorithm would silently ignore (§18-21). Recognizing exactly where a design's physical and financial realities can diverge — and inserting a durable, recoverable step at precisely that point — is the transferable skill this guide is really teaching.

---

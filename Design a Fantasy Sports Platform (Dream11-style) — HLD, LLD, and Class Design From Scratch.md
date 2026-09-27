# 1. What We Are Building

A fantasy sports platform in the style of Dream11: users create virtual teams by drafting real athletes from an upcoming real-world match under a fixed salary cap, join paid or free contests built around that match, and are ranked on a live leaderboard driven by the athletes' real match performance, converted into fantasy points by a scoring engine. Winners are paid out from the contest's prize pool once the match — and the scoring it depends on — is finalized.

This system sits at the intersection of four genuinely hard problems that don't normally show up together: real-money correctness (a wallet that must never be double-spent or double-paid), a hard real-time deadline (a contest must lock at kickoff, to the second, no exceptions), a massive fan-out write problem (one six-wicket over or one hat-trick must ripple into millions of already-submitted teams' scores within seconds), and a live-ranking problem (a leaderboard of those millions of teams must stay continuously, correctly sorted while it is being hammered by that same fan-out).

We will build this from first principles: define the domain model, design the contest-entry and wallet-debit paths so they are provably race-free, design the scoring fan-out and leaderboard update paths so they scale, and design contest locking and payout so they are exactly-once even across process crashes and retries.

---

# 2. Learning Objectives

By the end of this guide, you should be able to:

- Model a multi-stage lifecycle (draft team → join contest → match live → match finalized → payout) as an explicit state machine, and explain why each transition must be guarded.
- Design a lineup validator that enforces a salary cap and role-composition rules as pluggable, sport-specific strategies rather than hardcoded conditionals.
- Identify and fix the "contest overbooking" race — many users joining a fixed-size contest at once — using the same atomic-reservation family of techniques this series has used for seat booking and cash dispensing.
- Identify and fix wallet double-spend under concurrent contest joins, and explain why this is the same underlying problem as overbooking, solved with the same tool, applied to a different resource.
- Design a scoring fan-out pipeline that converts one real-world event (a wicket, a goal, a touchdown) into point updates across millions of fantasy teams without recomputing every team's full score from scratch on every event.
- Design a real-time leaderboard that stays correctly ranked under continuous high-volume score updates, without re-sorting the entire contest on every point change.
- Explain why contest locking at kickoff and prize payout after finalization both need an idempotency guarantee, and design both so a retried or duplicated trigger cannot lock twice or pay out twice.

---

# 3. Why This Matters (The Interview, Framed)

Interviewers reach for "design Dream11" or "design a fantasy sports app" because it looks like a CRUD product on the surface — pick some players, join a contest, see a leaderboard — but every one of those three steps hides a real distributed-systems problem once you ask "what happens at scale, under concurrency, with real money involved?"

> **Interviewer:** *"Walk me through what happens when a user hits 'Join Contest.' What could possibly go wrong?"*

A junior answer describes a form submission. A strong answer immediately identifies two independent race conditions living in that one click: the contest might have exactly one spot left and thousands of users might click "Join" in the same second (an overbooking race), and the same user's wallet balance might be read and debited by two concurrent requests from two devices (a double-spend race). Recognizing that "Join Contest" is actually two atomicity problems wearing one UI button is the entire signal this question is designed to surface.

The rest of the system rewards the same instinct: "how does the leaderboard update live" is really "how do you fan out one write to millions of readers without falling over," and "how do you pay out winners" is really "how do you guarantee an exactly-once financial transfer against a process that might crash mid-payout and get retried." We'll build the system so every one of these questions has a concrete, defensible answer instead of a hand-wave.

---

# 4. Recommended Technology Stack

| Layer | Choice | Why |
|---|---|---|
| API layer | REST + WebSocket (or SSE) | REST for team creation and contest joins; a push channel for live leaderboard and score updates, since polling millions of clients every few seconds does not scale. |
| Contest/lineup service | Java / Kotlin (Spring Boot) or Go | Strong typing suits the salary-cap and role-validation rule engine; both ecosystems have mature transactional-database drivers. |
| Wallet ledger | PostgreSQL (or any strongly-consistent RDBMS) | Real money demands ACID transactions and durable, auditable writes — this is not a place for eventual consistency. |
| Contest/team metadata | PostgreSQL | Relational integrity between users, teams, lineups, and contests; foreign keys catch a whole class of bugs for free. |
| Live scoring ingestion | Kafka (or equivalent log-based queue) | A durable, ordered, replayable log of raw match events is the only sane source of truth for a fan-out pipeline — replay is how you recover from a downstream bug without re-scraping the match. |
| Fantasy point computation | Stream processor (Flink / Kafka Streams) or a dedicated scoring service | Converts raw match events into per-player fantasy-point deltas exactly once per event, independent of how many teams drafted that player. |
| Leaderboard store | Redis (sorted sets) | A sorted set gives O(log N) rank updates and O(log N + K) range queries — exactly the two operations a live leaderboard needs, without re-sorting anything. |
| Cache | Redis | Player stats, contest metadata, and lineup snapshots are read far more often than written. |
| Object storage | S3-compatible | Player photos, team logos, downloadable contest reports. |
| Notification/push | FCM / APNS + a pub/sub fan-out layer | Score-change and contest-result push notifications at fantasy-sports scale are themselves a fan-out problem, layered on top of the leaderboard fan-out. |

---

# 5. Project Structure

```text
fantasy-sports/
├── api/
│   ├── TeamController.java
│   ├── ContestController.java
│   ├── WalletController.java
│   └── LeaderboardController.java
├── domain/
│   ├── User.java
│   ├── Player.java
│   ├── Match.java
│   ├── FantasyTeam.java
│   ├── LineupSlot.java
│   ├── Contest.java
│   ├── ContestEntry.java
│   └── WalletAccount.java
├── lineup/
│   ├── LineupValidator.java                  -- §17
│   ├── SalaryCapRule.java                    -- §17
│   ├── RoleCompositionRule.java               -- §17
│   └── SportRuleSet.java                     -- §17
├── contest/
│   ├── ContestLifecycle.java                 -- §13-14
│   ├── ContestState.java                     -- §13
│   ├── AtomicContestEntryService.java         -- §20-21
│   └── ContestLockScheduler.java             -- §32-33
├── wallet/
│   ├── WalletLedger.java                     -- §23-24
│   └── WalletTransaction.java                -- §23
├── scoring/
│   ├── MatchEventConsumer.java                -- §26-27
│   ├── FantasyPointCalculator.java            -- §26
│   └── ScoringRuleSet.java                    -- §26
├── leaderboard/
│   ├── LeaderboardService.java                -- §29-30
│   └── RankUpdatePublisher.java               -- §29
├── payout/
│   ├── PayoutProcessor.java                   -- §35-36
│   └── PayoutIdempotencyStore.java            -- §36
└── fraud/
    └── FraudSignalService.java                -- §40
```

---

# 6. Follow-up Question 1: What Are the Core Nouns Here, Before We Draw Any Boxes?

> **Interviewer:** *"Before you design anything, what are the core entities in a fantasy sports platform?"*

Naming them precisely up front avoids a common failure mode: conflating a "player" (a real athlete) with a "user" (a person using the app), or conflating a "contest" (a specific prize-pool competition) with a "match" (the underlying real-world sporting event that many contests can be built around). Get the nouns right and the rest of the design falls out of their relationships.

---

# 7. Functional Requirements

- Users can browse upcoming real-world matches and view the available contests built around each match.
- Users can create a fantasy team for a match by selecting a fixed number of real athletes, subject to a salary cap and role-composition rules (e.g., minimum wicketkeepers, minimum bowlers).
- Users can join one or more contests with a created team, provided each contest has an open slot and the user's wallet balance covers the entry fee.
- Contests automatically lock at the match's kickoff time; no team can be created, edited, or entered into a contest after lock.
- Once the match is live, athletes' real performance is converted into fantasy points in near-real-time, and every entered team's total score reflects those points within seconds.
- Users can view a live, continuously-updating leaderboard for any contest they've entered, ranked by total fantasy points, with deterministic tie-breaking.
- Once the match is finalized, prize money is distributed to winning entries according to the contest's payout structure, exactly once per entry.
- Users can view their wallet balance and full transaction history (deposits, entry-fee debits, winnings credits, withdrawals).
- Private contests support invite-only entry via a shareable code.

---

# 8. Non-Functional Requirements

- **Correctness over availability for money**: a wallet debit or credit must never be lost, duplicated, or applied against a stale balance, even under concurrent requests or a mid-operation crash.
- **Contest slot exactness**: a contest advertised as having N spots must never accept more than N entries, even under a burst of simultaneous join requests at deadline.
- **Low-latency fan-out**: a scoring event affecting a widely-drafted player must be reflected in every affected team's score within a few seconds, even when millions of teams have drafted that player.
- **Leaderboard read scalability**: leaderboard reads (rank + nearby entries) must stay fast even for a contest with tens of millions of entries, and even while writes are arriving continuously.
- **Exactly-once side effects**: contest locking and prize payout must each happen exactly once, regardless of retries, duplicate triggers, or process restarts.
- **Auditability**: every wallet movement must be reconstructable after the fact from an immutable ledger, for dispute resolution and regulatory reporting.

---

# 9. Follow-up Question 2: Why Model the Team/Contest Lifecycle as an Explicit State Machine Instead of a Status Column?

> **Interviewer:** *"Why not just add a `status` string column to the contest table and check it in application code wherever you need to?"*

A free-form status string lets any code path assign any value at any time — nothing stops a bug from re-opening a locked contest or joining a finalized one. An explicit state machine makes illegal transitions a compile-time or first-line-of-code impossibility rather than a hoped-for convention, and it gives every downstream reader (the entry service, the scoring pipeline, the payout processor) one unambiguous place to ask "is this contest still joinable right now?"

---

# 10. Identifying the Core Domain Entities

```text
User            -- a registered account with a wallet
Player          -- a real athlete participating in a real Match
Match           -- a real-world sporting fixture (two teams, a kickoff time)
FantasyTeam     -- one user's drafted lineup of Players for one Match
LineupSlot      -- one Player's assigned role-slot within a FantasyTeam
Contest         -- a prize-pool competition built around one Match
ContestEntry    -- one FantasyTeam entered into one Contest
WalletAccount   -- one user's real-money balance and transaction ledger
MatchEvent      -- one raw scoring-relevant event from the live Match (a wicket, a goal)
PlayerScore     -- one Player's cumulative fantasy points for one Match, derived from MatchEvents
```

The relationship that trips people up first: a `FantasyTeam` is created once per user per match, but that same team can be entered into *many* contests (a "mega contest" and a "head-to-head" contest built around the identical match). This is why `ContestEntry` is its own entity rather than a foreign key on `FantasyTeam` — the team is the athlete selection, the entry is the act of risking money on that selection inside one specific contest.

---

# 11. High-Level Architecture Overview

```text
                     ┌─────────────────┐
                     │   Client Apps    │
                     └────────┬─────────┘
                              │
                    ┌─────────▼──────────┐
                    │     API Gateway     │
                    └──┬────────┬────────┘
             ┌─────────┘        └──────────┐
   ┌─────────▼─────────┐          ┌────────▼─────────┐
   │  Team & Lineup Svc  │          │  Contest Entry Svc │
   │  (§17-18)           │          │  (§20-21)          │
   └─────────┬─────────┘          └────────┬─────────┘
             │                              │
             │                    ┌─────────▼─────────┐
             │                    │   Wallet Ledger DB  │
             │                    │   (§23-24)          │
             │                    └────────────────────┘
             │
   ┌─────────▼──────────────────────────────────────────┐
   │              Contest / Team Metadata DB              │
   └───────────────────────────────────────────────────────┘

  Live match feed provider
          │
  ┌───────▼────────┐     ┌──────────────────┐     ┌────────────────────┐
  │  Match Event Log │────▶│ Fantasy Point     │────▶│  Leaderboard Store  │
  │  (Kafka) (§26)   │     │ Calculator (§26-27)│     │  (Redis) (§29-30)   │
  └──────────────────┘     └──────────────────┘     └─────────┬──────────┘
                                                                │
                                                       ┌────────▼─────────┐
                                                       │  WebSocket / SSE   │
                                                       │  Fan-out to Clients│
                                                       └────────────────────┘
```

The split that matters most: the write path for *money and lineups* (top half — strongly consistent, relational) is architecturally separate from the write path for *live scores* (bottom half — a log-based, massively fanned-out pipeline). They only meet at the leaderboard, which reads contest-entry data once at join time and otherwise only consumes score deltas. Mixing these two paths into one database would force the money path to inherit the scoring path's write volume, or force the scoring path to inherit the money path's strict consistency latency — neither is acceptable.

---

# 12. Follow-up Question 3: How Do You Model the Contest Lifecycle, and Why Isn't "Open" and "Locked" Enough?

> **Interviewer:** *"What states does a contest actually go through, end to end?"*

"Open" and "locked" describes joining, but the contest keeps existing — and keeps needing correct behavior — long after it locks: it must accept live scoring updates while the match is in progress, stop accepting them once the match ends, and pay out exactly once after that. Collapsing all of that into two states forces every one of those later phases to be inferred from side information (is the match over? has payout run yet?) instead of being a first-class fact anyone can check.

---

# 13. The Contest Lifecycle as a State Machine

```text
   ┌────────┐  join opens  ┌────────┐  kickoff time   ┌────────┐  match ends  ┌─────────────┐  payout runs  ┌──────────┐
   │ CREATED │────────────▶│  OPEN   │───────────────▶│ LOCKED  │─────────────▶│ FINALIZING   │──────────────▶│ COMPLETED │
   └────────┘              └────────┘                 └────────┘              └─────────────┘               └──────────┘
                                                            │                        │
                                                            │ match abandoned/washed │
                                                            ▼                        ▼
                                                       ┌──────────┐            ┌───────────┐
                                                       │ CANCELLED │◀───────────│  DISPUTED  │
                                                       └──────────┘            └───────────┘
```

- **CREATED** — the contest exists but isn't yet visible for entry (used for scheduled pre-launch).
- **OPEN** — accepting `ContestEntry` creations, subject to slot availability and wallet balance.
- **LOCKED** — kickoff has passed; no new entries, no lineup edits on existing entries. Scoring updates still flow in and the leaderboard still updates live.
- **FINALIZING** — the match has ended; scores are settling (some sports have post-match score corrections, e.g., an umpire review overturning a decision minutes after play). Payout has not yet run.
- **COMPLETED** — payout has run exactly once; entries and final standings are now immutable history.
- **CANCELLED** — the match was abandoned or washed out before a result; every entry fee is refunded via the same wallet ledger used for debits.
- **DISPUTED** — a scoring or result dispute is under manual review; payout is held until resolved, then the contest returns to FINALIZING.

Every transition is one-directional except DISPUTED→FINALIZING, and every service in the system checks this state — never a boolean flag — before acting.

---

# 14. Implementing the State Pattern for Contest Lifecycle

```java
public enum ContestState {
    CREATED, OPEN, LOCKED, FINALIZING, DISPUTED, COMPLETED, CANCELLED
}

public class Contest {
    private final String contestId;
    private final String matchId;
    private ContestState state;
    private final int totalSlots;
    private int filledSlots;

    private static final Map<ContestState, Set<ContestState>> ALLOWED_TRANSITIONS = Map.of(
        ContestState.CREATED,    Set.of(ContestState.OPEN, ContestState.CANCELLED),
        ContestState.OPEN,       Set.of(ContestState.LOCKED, ContestState.CANCELLED),
        ContestState.LOCKED,     Set.of(ContestState.FINALIZING, ContestState.CANCELLED),
        ContestState.FINALIZING, Set.of(ContestState.DISPUTED, ContestState.COMPLETED),
        ContestState.DISPUTED,   Set.of(ContestState.FINALIZING)
    );

    public synchronized void transitionTo(ContestState next) {
        Set<ContestState> allowed = ALLOWED_TRANSITIONS.getOrDefault(state, Set.of());
        if (!allowed.contains(next)) {
            throw new IllegalStateException("Cannot go from " + state + " to " + next);
        }
        this.state = next;
    }

    public boolean isJoinable() {
        return state == ContestState.OPEN && filledSlots < totalSlots;
    }
}
```

Every write path — entry creation, lineup edits, scoring ingestion, payout — calls a narrow, purpose-built check (`isJoinable()`, `isAcceptingScoreUpdates()`, `isPayable()`) rather than comparing `state` directly, so the *meaning* of a state lives in exactly one place even though many services need to ask about it.

---

# 15. Class Diagram: The Contest Lifecycle Core

```text
┌────────────────────┐        ┌─────────────────────┐
│      Contest         │◀──────│    ContestEntry       │
│ ─────────────────── │  1   * │ ──────────────────── │
│ contestId            │        │ entryId               │
│ matchId               │        │ contestId             │
│ state: ContestState   │        │ teamId                │
│ totalSlots            │        │ userId                │
│ filledSlots           │        │ enteredAt             │
│ entryFeeCents         │        │ finalRank             │
│ prizeStructure        │        │ prizeCents            │
│ transitionTo()        │        └─────────────────────┘
│ isJoinable()          │
└──────────┬───────────┘
           │
           │ built around
           ▼
┌────────────────────┐        ┌─────────────────────┐
│       Match          │◀──────│      FantasyTeam       │
│ ─────────────────── │  1   * │ ──────────────────── │
│ matchId               │        │ teamId                │
│ kickoffTime           │        │ userId                │
│ status                │        │ matchId               │
└────────────────────┘        │ lineup: List<LineupSlot>│
                                └─────────────────────┘
```

---

# 16. Follow-up Question 4: How Do You Enforce a Salary Cap and Role Composition Without Hardcoding One Sport's Rules?

> **Interviewer:** *"Cricket needs a wicketkeeper and bowlers; football needs a goalkeeper and a formation. How do you avoid an `if (sport == CRICKET)` mess?"*

Every sport shares the same *shape* of rule — a total budget constraint plus a set of minimum/maximum counts per role — even though the actual numbers differ per sport. That shape is exactly what the Strategy pattern is for: one `LineupValidator` interface, one `SportRuleSet` per sport supplying the numbers, and zero sport-specific conditionals anywhere in the validation logic itself.

---

# 17. Team Selection as a Pluggable Rule Set

```java
public interface SportRuleSet {
    int squadSize();
    long salaryCapCents();
    Map<PlayerRole, IntRange> roleCountConstraints();   // e.g. WICKETKEEPER -> [1,1]
    int maxPlayersFromOneRealTeam();                    // avoid an all-one-side lineup
}

public class CricketRuleSet implements SportRuleSet {
    public int squadSize() { return 11; }
    public long salaryCapCents() { return 10_000_00; }
    public Map<PlayerRole, IntRange> roleCountConstraints() {
        return Map.of(
            PlayerRole.WICKETKEEPER, new IntRange(1, 1),
            PlayerRole.BATSMAN,      new IntRange(3, 6),
            PlayerRole.ALL_ROUNDER,  new IntRange(1, 4),
            PlayerRole.BOWLER,       new IntRange(3, 6)
        );
    }
    public int maxPlayersFromOneRealTeam() { return 7; }
}
```

---

# 18. Implementing the Lineup Validator

```java
public class LineupValidator {
    private final SportRuleSet rules;

    public ValidationResult validate(List<LineupSlot> lineup) {
        if (lineup.size() != rules.squadSize()) {
            return ValidationResult.fail("Squad must have exactly " + rules.squadSize() + " players");
        }
        long totalCost = lineup.stream().mapToLong(s -> s.player().priceCents()).sum();
        if (totalCost > rules.salaryCapCents()) {
            return ValidationResult.fail("Over salary cap by " + (totalCost - rules.salaryCapCents()) + " cents");
        }
        Map<PlayerRole, Long> roleCounts = lineup.stream()
            .collect(Collectors.groupingBy(s -> s.player().role(), Collectors.counting()));
        for (var entry : rules.roleCountConstraints().entrySet()) {
            long count = roleCounts.getOrDefault(entry.getKey(), 0L);
            if (!entry.getValue().contains(count)) {
                return ValidationResult.fail(entry.getKey() + " count " + count + " outside allowed range");
            }
        }
        Map<String, Long> perRealTeamCounts = lineup.stream()
            .collect(Collectors.groupingBy(s -> s.player().realTeamId(), Collectors.counting()));
        if (perRealTeamCounts.values().stream().anyMatch(c -> c > rules.maxPlayersFromOneRealTeam())) {
            return ValidationResult.fail("Too many players from a single real-world team");
        }
        return ValidationResult.ok();
    }
}
```

A new sport means writing one new `SportRuleSet` implementation — the validator, the API contract, and every caller stay untouched. This is the Open/Closed Principle earning its keep, not just a textbook label.

---

# 19. Follow-up Question 5: A Contest Has Exactly 10,000 Spots and 40,000 Users Click "Join" in the Final Second Before Lock — What Happens?

> **Interviewer:** *"How do you make sure exactly 10,000 — not 10,004, not 9,998 — entries get created?"*

This is the identical structural problem this series has already solved twice: the BookMyShow guide's seat-double-booking race, and the ATM guide's account-overdraft race. A naive "check remaining slots, then insert an entry" is a check-then-act race — under concurrency, many requests can all read "1 slot left" before any of them writes, and all of them proceed. The fix is the same family of tools applied to a new resource: a fixed pool of contest slots instead of seats or cash.

---

# 20. Contest Entry as an Atomic Slot Reservation

The safest and simplest fix here is an **atomic decrement-with-floor** — a single database statement that both checks and reserves a slot in one indivisible operation, so no interleaving of two requests can ever observe a stale count:

```sql
UPDATE contest
SET filled_slots = filled_slots + 1
WHERE contest_id = :contestId
  AND state = 'OPEN'
  AND filled_slots < total_slots;
-- if this UPDATE affects 0 rows, the contest was full (or not open) -- reject the join
-- if it affects 1 row, this request legitimately won the last slot -- proceed to create the ContestEntry
```

This works because the `WHERE filled_slots < total_slots` condition and the `+1` write happen inside one atomic statement evaluated by the database against the *current* row — there is no gap between reading the count and writing it for another concurrent transaction to land in. Exactly the same principle as the ATM guide's `AtomicInteger.updateAndGet` and BookMyShow's `SELECT ... FOR UPDATE`, expressed as a single conditional `UPDATE`.

---

# 21. Implementing the Full Atomic Contest-Entry Service

```java
public class AtomicContestEntryService {
    public EntryResult joinContest(String contestId, String userId, String teamId, long entryFeeCents) {
        int rowsUpdated = jdbcTemplate.update(
            "UPDATE contest SET filled_slots = filled_slots + 1 " +
            "WHERE contest_id = ? AND state = 'OPEN' AND filled_slots < total_slots",
            contestId
        );
        if (rowsUpdated == 0) {
            return EntryResult.failed("Contest is full or no longer open");
        }
        WalletDebitResult debit = walletLedger.debit(userId, entryFeeCents, "contest_entry:" + contestId);
        if (!debit.success()) {
            jdbcTemplate.update(
                "UPDATE contest SET filled_slots = filled_slots - 1 WHERE contest_id = ?", contestId
            );
            return EntryResult.failed("Insufficient wallet balance");
        }
        contestEntryRepository.save(new ContestEntry(contestId, userId, teamId, Instant.now()));
        return EntryResult.success();
    }
}
```

Notice the slot is reserved *before* the wallet is touched, and released back if the debit fails — the same reserve-then-confirm shape the ATM guide used for cash, applied here to a contest slot instead of a cassette note. The next section handles the wallet debit itself, which has its own, independent race to close.

---

# 22. Follow-up Question 6: The Same User Joins Two Contests From Two Devices at the Same Instant — Can Their Wallet Go Negative?

> **Interviewer:** *"Balance is $500. Two devices each try to join a $400 contest at the same moment. What stops both from succeeding?"*

This is a second, independent race hiding in the same button click — even with the contest-slot race fixed, a naive wallet debit (`read balance, check sufficient, write new balance`) is itself a check-then-act race. Two concurrent debits can each read the pre-debit $500, each see "sufficient," and each write a new balance computed from that same stale $500 — leaving the account at $100 instead of correctly being rejected once at -$300. This is not a variant of the slot race — it's the *exact same class of bug*, applied to a different piece of shared state, and it needs the exact same style of fix.

---

# 23. Wallet Ledger and Preventing Double-Spend

The wallet balance is never trusted as a mutable field to read-then-write. Instead, every debit is an atomic conditional update, guarded by the balance itself, mirroring §20's `filled_slots` guard exactly:

```sql
UPDATE wallet_account
SET balance_cents = balance_cents - :amountCents
WHERE user_id = :userId
  AND balance_cents >= :amountCents;
-- 0 rows affected -> insufficient funds, reject
-- 1 row affected   -> debit succeeded atomically
```

Every debit and credit is additionally recorded as an immutable row in a `wallet_transaction` ledger table — the `balance_cents` column is a cached, always-recomputable projection of that ledger, never the sole source of truth. If the cached balance and the ledger's sum ever disagree, the ledger wins and the balance is repaired from it; this is what makes the system auditable rather than merely "usually correct."

---

# 24. Implementing the Atomic Wallet Debit

```java
public class WalletLedger {
    public WalletDebitResult debit(String userId, long amountCents, String reference) {
        int rowsUpdated = jdbcTemplate.update(
            "UPDATE wallet_account SET balance_cents = balance_cents - ? " +
            "WHERE user_id = ? AND balance_cents >= ?",
            amountCents, userId, amountCents
        );
        if (rowsUpdated == 0) {
            return WalletDebitResult.insufficientFunds();
        }
        transactionRepository.save(new WalletTransaction(
            userId, -amountCents, reference, TransactionType.DEBIT, Instant.now()
        ));
        return WalletDebitResult.success();
    }

    public void credit(String userId, long amountCents, String reference) {
        jdbcTemplate.update(
            "UPDATE wallet_account SET balance_cents = balance_cents + ? WHERE user_id = ?",
            amountCents, userId
        );
        transactionRepository.save(new WalletTransaction(
            userId, amountCents, reference, TransactionType.CREDIT, Instant.now()
        ));
    }
}
```

Credits never need the conditional guard — there's no "insufficient" case for adding money — but they still write an immutable ledger row, for the same audit reason. Combined with §21's atomic slot reservation, a "Join Contest" click is now backed by two independently race-free atomic operations instead of one racy button press.

---

# 25. Follow-up Question 7: A Popular Player Hits a Match-Winning Six — How Does That One Event Reach Millions of Teams' Scores Within Seconds?

> **Interviewer:** *"Ten million teams have drafted this player. One event happens. How do you avoid ten million database writes, or ten million re-reads of the full match state?"*

The naive approach — on every match event, recompute every affected team's total score from scratch by re-reading all of that team's players' full stats — does an amount of work proportional to (events) × (teams containing the affected player), which is untenable at fantasy-sports scale. The fix is to separate "what changed" from "who is affected," and only touch the second set with a small, precomputed *delta*, never a full recomputation.

---

# 26. Live Scoring Ingestion and the Fan-out Pipeline

```text
 Live match feed          Kafka topic           Fantasy Point            Kafka topic            Leaderboard
 provider                 "match-events"         Calculator               "player-score-deltas"   Update Consumer
 ┌─────────────┐         ┌─────────────┐        ┌─────────────────┐      ┌─────────────────┐     ┌───────────────┐
 │ "Player X    │────────▶│  raw event   │───────▶│ apply scoring     │─────▶│ PlayerX: +8 pts   │────▶│ for every team  │
 │  hits six"   │         │  (ordered,   │        │ rules -> point    │      │  (delta, not      │     │ that drafted    │
 │              │         │  durable)    │        │ delta for Player X│      │   absolute score) │     │ Player X, add   │
 └─────────────┘         └─────────────┘        └─────────────────┘      └─────────────────┘     │ +8 to their     │
                                                                                                     │ cached total    │
                                                                                                     └───────────────┘
```

The critical design choice: the `FantasyPointCalculator` computes **one delta per player per event** — not one update per affected team. "Which teams drafted Player X" is answered by a precomputed, indexed lookup (`player_id -> [team_id, ...]`), built once when each team is created, not recomputed per event. Applying the delta to every affected team's cached total is then a bulk, embarrassingly-parallel fan-out — the expensive "who is affected" join happened once at team-creation time, not once per event.

---

# 27. Implementing the Scoring Event Pipeline

```java
public class FantasyPointCalculator {
    public void onMatchEvent(MatchEvent event) {
        ScoringRuleSet rules = scoringRuleSetFor(event.sport());
        int pointDelta = rules.pointsFor(event.eventType(), event.magnitude());
        PlayerScoreDelta delta = new PlayerScoreDelta(event.playerId(), event.matchId(), pointDelta, event.eventId());
        scoreDeltaTopic.publish(delta);
    }
}

public class LeaderboardUpdateConsumer {
    public void onPlayerScoreDelta(PlayerScoreDelta delta) {
        List<String> affectedTeamIds = teamsByPlayerIndex.lookup(delta.playerId(), delta.matchId());
        for (String teamId : affectedTeamIds) {
            List<String> contestIds = entriesByTeamIndex.lookup(teamId);
            for (String contestId : contestIds) {
                leaderboardService.incrementScore(contestId, teamId, delta.pointDelta());  // §30
            }
        }
    }
}
```

`event.eventId()` is carried through end to end specifically so this pipeline can be made idempotent (§29's reconciliation pattern generalizes here too) — a redelivered Kafka message must add its point delta exactly once, never twice, which means the leaderboard increment itself must be de-duplicated per `eventId` per team, not just per event.

---

# 28. Follow-up Question 8: A Contest Has 20 Million Entries — How Do You Keep It Correctly Ranked Without Re-Sorting on Every Point Change?

> **Interviewer:** *"Every few seconds thousands of point deltas land. How does 'show me rank 1 through 100' stay fast?"*

Re-sorting 20 million rows on every incoming delta is obviously wrong, but so is the subtler mistake of running an `ORDER BY score DESC LIMIT 100` SQL query against a relational table on every leaderboard read — that query re-sorts a meaningful fraction of the table on every single read, and reads vastly outnumber writes on a leaderboard. What's needed is a data structure that keeps itself incrementally sorted as it's written, so a read is just "look at the front" rather than "re-derive the order."

---

# 29. Real-Time Leaderboard Design

A **sorted set** (Redis `ZSET`, or an equivalent skip-list-backed structure) is exactly this data structure: every member (a `teamId`) has a score, the structure is always kept in score order internally, and both the update and the range-read operations are O(log N) — neither one requires touching the other N-1 members.

```text
ZINCRBY contest:{contestId}:leaderboard  8  team:{teamId}     -- O(log N) incremental update
ZREVRANGE contest:{contestId}:leaderboard 0 99 WITHSCORES     -- O(log N + 100) top-100 read
ZREVRANK contest:{contestId}:leaderboard team:{teamId}         -- O(log N) "what's my rank" read
```

Deterministic tie-breaking (two teams on the exact same point total) is handled by encoding a tiebreaker into the score itself rather than leaving ties to be broken arbitrarily by the data structure — e.g. `score = (points * 1_000_000) - entryTimestampSeconds`, so earlier entries rank higher on a tie, and the comparison stays a single numeric compare with no secondary sort step.

---

# 30. Implementing the Leaderboard Update Path

```java
public class LeaderboardService {
    public void incrementScore(String contestId, String teamId, int pointDelta) {
        String key = "contest:" + contestId + ":leaderboard";
        redis.zIncrBy(key, pointDelta, "team:" + teamId);
        rankUpdatePublisher.publishDelta(contestId, teamId, pointDelta);  // fan out to live WebSocket subscribers
    }

    public List<LeaderboardEntry> topN(String contestId, int n) {
        String key = "contest:" + contestId + ":leaderboard";
        return redis.zRevRangeWithScores(key, 0, n - 1).stream()
            .map(LeaderboardEntry::fromRedisTuple)
            .toList();
    }

    public long rankOf(String contestId, String teamId) {
        String key = "contest:" + contestId + ":leaderboard";
        return redis.zRevRank(key, "team:" + teamId);
    }
}
```

Clients subscribed to a contest's leaderboard receive only the small, incremental `rankUpdatePublisher` delta over a WebSocket, not a re-fetch of the whole leaderboard — the same "send only what changed" principle from §26-27 applied one layer further downstream, all the way to the browser.

---

# 31. Class Diagram: Scoring and Leaderboard Flow

```text
┌─────────────┐      ┌────────────────────┐      ┌─────────────────────┐      ┌────────────────────┐
│  MatchEvent   │─────▶│ FantasyPointCalc.   │─────▶│  PlayerScoreDelta     │─────▶│ LeaderboardUpdate    │
│ ───────────  │      │ ──────────────────│      │ ───────────────────│      │ Consumer             │
│ eventId       │      │ pointsFor()         │      │ playerId              │      │ ───────────────────│
│ playerId      │      │                     │      │ matchId               │      │ onPlayerScoreDelta() │
│ matchId       │      └────────────────────┘      │ pointDelta            │      └──────────┬──────────┘
│ eventType     │                                    │ eventId               │                 │
└─────────────┘                                    └─────────────────────┘                 ▼
                                                                                   ┌────────────────────┐
                                                                                   │  LeaderboardService  │
                                                                                   │ ──────────────────  │
                                                                                   │ incrementScore()      │
                                                                                   │ topN()                │
                                                                                   │ rankOf()              │
                                                                                   └────────────────────┘
```

---

# 32. Follow-up Question 9: What Happens If a Match Kicks Off a Few Seconds Late — Does Contest Lock Timing Drift, Too?

> **Interviewer:** *"Kickoff is scheduled for 3:00:00 PM, but the toss/coin-flip runs long and play actually starts at 3:00:45. When exactly does the contest lock?"*

Locking must happen at the *scheduled* kickoff time, not the actual one — locking late (waiting for "actual" kickoff) would let users edit lineups after they've already seen live play begin and gained an unfair information advantage, which defeats the entire premise of a fair contest. This means contest locking cannot be a reactive response to a match-status change; it must be a proactive, precisely-scheduled action.

---

# 33. Contest Locking and Late-Join Prevention

Every contest's lock is scheduled at creation time, against the match's fixed `kickoffTime`, rather than being computed lazily whenever someone happens to ask "is this locked yet":

```java
public class ContestLockScheduler {
    public void scheduleLock(Contest contest, Instant kickoffTime) {
        delayedJobQueue.schedule(kickoffTime, () -> lockContest(contest.contestId()));
    }

    public void lockContest(String contestId) {
        int rowsUpdated = jdbcTemplate.update(
            "UPDATE contest SET state = 'LOCKED' WHERE contest_id = ? AND state = 'OPEN'",
            contestId
        );
        // 0 rows -> already locked (job re-ran, or a race with manual lock) -- fine, idempotent no-op
    }
}
```

Two details make this robust rather than merely "usually on time." First, the `WHERE state = 'OPEN'` guard makes `lockContest` idempotent — a scheduler retry, a duplicate trigger, or a crash-and-replay of the delayed job can call it any number of times and the contest still locks exactly once, in the database sense that matters (its state only ever transitions once). Second, and just as important: **every read on the join path independently re-checks `isJoinable()` at request time** (§21's `AND state = 'OPEN'` in the same atomic `UPDATE`) — so even if the scheduled lock job itself were delayed by a few hundred milliseconds under load, a join request arriving after the *actual* kickoff moment is still safe, because it isn't relying on the scheduler having already flipped the flag. The scheduler is what makes locking happen close to on time; the atomic guard on every join is what makes it *correct* regardless of exactly when the scheduler fires.

---

# 34. Follow-up Question 10: The Match Is Abandoned Mid-Innings Due to Rain — What Happens to Every Entry Fee and Every Lineup?

> **Interviewer:** *"A match is called off with no result. What's the correct behavior?"*

A contest built around a match with no official result cannot fairly be scored or paid out — there is no "final" performance to rank teams by. The only fair resolution is a full refund of every entry fee, which is why CANCELLED exists as its own terminal state (§13) rather than being treated as an edge case of FINALIZING: it needs entirely different downstream behavior (refund every entry, pay out none) than a normal completion.

---

# 35. Offline/Degraded Mode and Fail-Safe Defaults

When the live match-feed provider itself becomes unreachable mid-match, the system must fail toward safety, not toward availability at any cost:

```text
Live feed unreachable for > threshold seconds
        │
        ▼
Pause fantasy-point ingestion (do NOT guess scores from a stale feed)
        │
        ▼
Leaderboard freezes at last-known-good scores (labeled "provisional" in the UI)
        │
        ▼
Contest stays LOCKED, cannot transition to FINALIZING until feed resumes
   and reconciles against an authoritative post-match scorecard
```

This mirrors the ATM guide's offline/degraded-mode principle exactly: when the source of truth is unreachable, refuse to guess and freeze in a clearly-labeled provisional state, rather than silently computing wrong numbers that look authoritative.

---

# 36. Follow-up Question 11: The Match Ends, Payout Runs, and Then the Payout Job Crashes Halfway Through 20 Million Entries — What Happens on Restart?

> **Interviewer:** *"Payout is processing entry 8 million of 20 million when the process dies. It restarts. How do you avoid double-paying the first 8 million?"*

Restarting a payout job from the beginning is only safe if every individual payout is idempotent — otherwise the restart pays 8 million winners twice. The fix is the same idempotency-key pattern used throughout this series' payment-adjacent flows: record a durable, unique fact *before* the money moves, and check for that fact before moving money again.

---

# 37. Prize Distribution as an Idempotent Operation

```java
public class PayoutProcessor {
    public void runPayoutForContest(String contestId) {
        List<ContestEntry> rankedEntries = contestEntryRepository.findRankedByContest(contestId);
        PrizeStructure prizes = prizeStructureRepository.findByContest(contestId);
        for (ContestEntry entry : rankedEntries) {
            long prizeCents = prizes.prizeForRank(entry.finalRank());
            if (prizeCents <= 0) continue;
            String idempotencyKey = "payout:" + contestId + ":" + entry.entryId();
            if (payoutIdempotencyStore.alreadyProcessed(idempotencyKey)) {
                continue;  // this entry was already paid in a prior, interrupted run
            }
            walletLedger.credit(entry.userId(), prizeCents, idempotencyKey);
            payoutIdempotencyStore.markProcessed(idempotencyKey);
        }
        contestLifecycle.transitionTo(contestId, ContestState.COMPLETED);
    }
}
```

The `idempotencyKey` — deterministically derived from `contestId` and `entryId`, never a random value — is what makes "check, then act" safe here even though it's the same two-step shape this guide has flagged as dangerous everywhere else. The difference is that `payoutIdempotencyStore.markProcessed` and the credit's own ledger row are the durable record being checked, not a soon-to-be-stale in-memory count — so a crash between crediting and marking-processed is the only genuinely dangerous window, and it's closed by making `markProcessed` part of the same database transaction as the credit itself.

---

# 38. Implementing Payout Idempotency Storage

```java
public class PayoutIdempotencyStore {
    public boolean alreadyProcessed(String idempotencyKey) {
        return jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM payout_ledger WHERE idempotency_key = ?", Integer.class, idempotencyKey
        ) > 0;
    }

    public void markProcessed(String idempotencyKey) {
        jdbcTemplate.update(
            "INSERT INTO payout_ledger (idempotency_key, processed_at) VALUES (?, ?) " +
            "ON CONFLICT (idempotency_key) DO NOTHING",
            idempotencyKey, Instant.now()
        );
    }
}
```

The `ON CONFLICT DO NOTHING` unique-constraint guard is the actual safety net — even if two payout-job instances somehow ran concurrently over the same contest (which the `COMPLETED` state transition is meant to prevent), the database's own uniqueness constraint on `idempotency_key` makes a duplicate insert physically impossible, not just unlikely.

---

# 39. Follow-up Question 12: How Do You Support Multiple Transaction Types (Entry Fees, Winnings, Deposits, Withdrawals, Refunds) Without a Wallet God-Object?

> **Interviewer:** *"Your wallet needs to handle five different kinds of money movement, each with its own validation rules. How do you keep that from becoming an unmaintainable pile of conditionals?"*

Each transaction type shares the same underlying primitive (an atomic, ledgered balance mutation) but differs in *when* it's allowed and what triggers it — a withdrawal needs KYC verification and a cooldown check that an entry-fee debit doesn't. This is the same Strategy-pattern shape as §17's sport rules: one common mutation primitive, pluggable pre-conditions per transaction type.

---

# 40. Implementing Pluggable Transaction Strategies

```java
public interface TransactionStrategy {
    ValidationResult preValidate(String userId, long amountCents);
    TransactionType type();
}

public class WithdrawalStrategy implements TransactionStrategy {
    public ValidationResult preValidate(String userId, long amountCents) {
        if (!kycService.isVerified(userId)) return ValidationResult.fail("KYC not verified");
        if (withdrawalCooldown.isActive(userId)) return ValidationResult.fail("Cooldown active");
        return ValidationResult.ok();
    }
    public TransactionType type() { return TransactionType.WITHDRAWAL; }
}

public class WalletTransactionService {
    public TransactionResult execute(String userId, long amountCents, TransactionStrategy strategy) {
        ValidationResult validation = strategy.preValidate(userId, amountCents);
        if (!validation.isValid()) return TransactionResult.rejected(validation.reason());
        return switch (strategy.type()) {
            case WITHDRAWAL, ENTRY_FEE -> TransactionResult.from(walletLedger.debit(userId, amountCents, strategy.type().name()));
            case WINNINGS, DEPOSIT, REFUND -> { walletLedger.credit(userId, amountCents, strategy.type().name()); yield TransactionResult.success(); }
        };
    }
}
```

Adding a new transaction type — say, a promotional bonus credit with its own eligibility rule — means writing one new `TransactionStrategy`, not touching `WalletTransactionService` or the atomic debit/credit primitives underneath it at all.

---

# 41. Deposit Handling and Provisional Crediting

A deposit initiated through a third-party payment gateway is not instantaneous or guaranteed — the gateway's own confirmation webhook can arrive late, arrive twice, or never arrive at all. Deposits therefore go through their own small state machine (`INITIATED → CONFIRMED` or `INITIATED → FAILED`), and the wallet is only credited on `CONFIRMED`, keyed by the gateway's own transaction id as the idempotency key — a duplicate webhook delivery hits the same `ON CONFLICT DO NOTHING` guard from §38, reusing the exact same idempotency mechanism built for payouts, applied to money coming in rather than money going out.

---

# 42. Capacity Estimation: Fan-out Volume and Leaderboard Read Load

Assume a marquee match: 10 million contest entries across all contests combined, drafting from a pool of 22 real players, and roughly 300 scoring-relevant events over a 3-hour match.

- **Fan-out writes**: worst case, a single popular player is drafted by ~40% of all entries. 300 events × (10M × 0.4) potentially-affected entries ≈ 1.2 billion leaderboard increments over 3 hours ≈ **~110,000 increments/second at peak** (events cluster, so peak is far above the flat average). A Redis `ZINCRBY` comfortably handles this rate; the design's job is making sure the fan-out *lookup* (which teams drafted this player) is an O(1) indexed read, not a scan, since that lookup runs once per affected entry per event.
- **Leaderboard reads**: assume 5 million concurrently-watching users each polling or subscribing to rank updates roughly once every few seconds. Pushed deltas over an already-open WebSocket (§30) cost far less than 5 million independent poll requests hitting `topN`/`rankOf` every few seconds — this is the concrete reason a push channel is a requirement, not a nice-to-have, once concurrent viewership crosses roughly the low hundreds of thousands.
- **Wallet writes**: entry-fee debits and payout credits are comparatively tiny in volume (one write per entry, one per payout) and comfortably fit on a conventional relational database — the money path was never the throughput bottleneck; the scoring fan-out always is.

---

# 43. Follow-up Question 13: How Do You Support a Private, Invite-Only Contest Among a Group of Friends?

> **Interviewer:** *"Add private contests to the design — what actually changes?"*

Surprisingly little of the core machinery changes: a private contest is still a `Contest` in every mechanical sense — it still uses the same atomic slot-reservation (§20-21), the same lifecycle state machine (§13), the same leaderboard (§29-30). The only real addition is a second, narrower gate in front of the existing join path: possession of a valid invite code, checked before the atomic reservation ever runs, so the two checks compose rather than duplicate each other.

---

# 44. Private Contests and Invite-Code Handling

```java
public class PrivateContestEntryService {
    public EntryResult joinPrivateContest(String contestId, String userId, String teamId,
                                            long entryFeeCents, String suppliedInviteCode) {
        String actualCode = inviteCodeRepository.findByContest(contestId);
        if (!MessageDigest.isEqual(actualCode.getBytes(UTF_8), suppliedInviteCode.getBytes(UTF_8))) {
            return EntryResult.failed("Invalid invite code");
        }
        return atomicContestEntryService.joinContest(contestId, userId, teamId, entryFeeCents);  // §21, unchanged
    }
}
```

`MessageDigest.isEqual` performs a constant-time comparison rather than a plain `.equals()` — a private contest's invite code is a lightweight access-control secret, and a naive string comparison leaks timing information about how many leading characters matched, which is exactly the kind of small omission that turns into a guessable-code vulnerability at scale. The join logic itself delegates straight back to §21's already-correct, already-race-free atomic reservation — this is the Open/Closed Principle again: extending behavior (an extra gate) without modifying the thing being extended.

---

# 45. Follow-up Question 14: How Do You Stop One Person From Registering 500 Accounts to Enter the Same Head-to-Head Contest Against Themselves?

> **Interviewer:** *"A user could create many accounts to guarantee a win in a small contest, or to launder money between their own wallets. What's the defense?"*

This is a fraud-detection problem layered on top of the correctness machinery already built, not a replacement for it — even a perfect anti-fraud system still needs the atomic slot reservation and wallet debit underneath it to be correct. The defense is a set of signals evaluated *before* an entry is allowed to proceed, not a retroactive cleanup after the fact.

---

# 46. Fraud Signal Detection

```java
public class FraudSignalService {
    public FraudCheckResult evaluate(String userId, String contestId, DeviceFingerprint device) {
        if (accountsSharingDevice(device).size() > MAX_ACCOUNTS_PER_DEVICE) {
            return FraudCheckResult.flag("Multiple accounts on one device");
        }
        if (accountsSharingDevice(device).stream().anyMatch(otherUserId ->
                contestEntryRepository.hasEntry(contestId, otherUserId))) {
            return FraudCheckResult.block("Linked account already entered this contest");
        }
        if (walletFundingSourceOverlap(userId, contestId) > FUNDING_OVERLAP_THRESHOLD) {
            return FraudCheckResult.flag("Shared funding source with co-entrants");
        }
        return FraudCheckResult.clear();
    }
}
```

A `block` result stops the entry outright (e.g., a confirmed linked account already in the same head-to-head); a `flag` result lets the entry proceed but queues it for manual or automated review — the distinction matters because false positives on a hard block directly cost legitimate users money and trust, so only the highest-confidence signals are allowed to block synchronously.

---

# 47. Full Worked Example: One Contest Traced End to End

1. **Team creation** — a user drafts 11 cricket players for tomorrow's match, spending $98.50 of a $100 salary cap; `LineupValidator` (§18) confirms role counts and real-team caps are satisfied.
2. **Contest join** — the user joins a ₹50-entry, 10,000-slot mega contest. The atomic `UPDATE ... WHERE filled_slots < total_slots` (§20) succeeds at slot 9,847; the wallet debit (§24) succeeds against a sufficient balance; a `ContestEntry` row is created.
3. **Kickoff** — the scheduler's delayed job (§33) flips the contest to `LOCKED` at the scheduled time; a join request arriving 40ms later independently fails the same atomic guard regardless of whether the scheduled job had already fired.
4. **Live play** — a wicket falls. `MatchEventConsumer` computes a fantasy-point delta for the bowler (§27), publishes it, and `LeaderboardUpdateConsumer` looks up every team containing that bowler (an O(1) indexed lookup) and increments each one's score in the Redis sorted set (§30) — including our user's team, if they drafted that bowler.
5. **Live viewing** — the user's app receives the rank-delta push over its open WebSocket subscription and updates their displayed rank without polling or re-fetching the leaderboard.
6. **Match ends** — the contest transitions to `FINALIZING`; a brief scoring-correction window closes with no disputes; it transitions to `COMPLETED`-pending-payout.
7. **Payout** — `PayoutProcessor` (§37) walks ranked entries, finds our user finished 812th (out of a paying threshold of 2,000), credits their wallet with the rank-812 prize amount, records the idempotency key, and moves on — a mid-run crash and restart at this exact point would skip our user's entry on retry, since `alreadyProcessed` would already be `true`.
8. **Wallet history** — the user's transaction history shows the ₹50 debit at join time and the prize credit at payout time as two separate, immutable, individually-timestamped ledger rows — never a single overwritten balance.

---

# 48. Final Architecture Diagram

```text
┌────────────┐   ┌──────────────┐   ┌───────────────────┐        ┌────────────────┐
│ Client Apps │──▶│ API Gateway   │──▶│ Team/Lineup Svc      │──▶│ Contest/Team DB  │
└────────────┘   └──────────────┘   │ (§17-18)             │   └────────────────┘
                          │           └───────────────────┘
                          ▼
                  ┌───────────────────┐      ┌──────────────────┐
                  │ Contest Entry Svc    │──▶│  Wallet Ledger DB   │
                  │ (atomic §20-21,       │   │  (§23-24, §37-38)   │
                  │  fraud gate §46)      │   └──────────────────┘
                  └───────────────────┘
                          │
                          ▼
                  ┌───────────────────┐
                  │ Contest Lock          │
                  │ Scheduler (§33)       │
                  └───────────────────┘

  Live match feed ──▶ Match Event Log (Kafka) ──▶ Fantasy Point Calculator (§26-27)
                                                          │
                                                          ▼
                                          Player-Score-Delta Topic (Kafka)
                                                          │
                                                          ▼
                                          Leaderboard Update Consumer (§27, §30)
                                                          │
                                                          ▼
                                          Redis Sorted Sets (per contest) (§29-30)
                                                          │
                                                          ▼
                                          WebSocket / SSE Fan-out ──▶ Client Apps

  Match ends ──▶ Payout Processor (§37-38, idempotent) ──▶ Wallet Ledger DB
```

---

# 49. Design Patterns Used Throughout This Guide

- **State** — `ContestState` (§13-14) and the deposit-confirmation state machine (§41) make illegal transitions structurally impossible.
- **Strategy** — `SportRuleSet` (§17), `ScoringRuleSet` (§26), and `TransactionStrategy` (§40) each isolate one axis of variation behind a shared interface.
- **Template Method** (implicit) — the join-contest flow (reserve slot → debit wallet → create entry, §21) and the withdrawal flow (pre-validate → debit → ledger, §40) each follow a fixed skeleton with pluggable steps.
- **Observer** — `RankUpdatePublisher` (§30) and the WebSocket fan-out layer notify subscribed clients of leaderboard changes without those clients polling.
- **Idempotent Command** — `PayoutProcessor` (§37-38) and deposit-webhook handling (§41) both key a durable "already done" fact off a deterministic identifier, making retries safe by construction.
- **Facade** — `AtomicContestEntryService` (§21) presents "join this contest" as one call while coordinating two independently atomic subsystems (slot reservation and wallet debit) underneath.

---

# 50. SOLID Principles Applied

- **Single Responsibility** — `LineupValidator` only validates; it never touches persistence, wallets, or contest state. `PayoutProcessor` only pays out; it never computes scores.
- **Open/Closed** — adding a new sport (§17) or a new transaction type (§40) or an additional fraud signal (§46) never requires modifying the classes that already work.
- **Liskov Substitution** — any `SportRuleSet` or `TransactionStrategy` implementation can be swapped in without the calling code needing to know which concrete implementation it received.
- **Interface Segregation** — `TransactionStrategy` exposes only `preValidate()` and `type()`; it does not force every implementation to also implement unrelated wallet-reporting or KYC concerns.
- **Dependency Inversion** — `AtomicContestEntryService` depends on `WalletLedger` as an interface/collaborator, not a concrete SQL implementation, so the wallet backing store could change without touching contest-entry logic.

---

# 51. Common Mistakes When Building This Yourself

- **Treating "join contest" as a single operation** instead of recognizing it hides two independent races (slot reservation and wallet debit) that each need their own atomic guard (§20-24).
- **Recomputing full team scores from scratch on every match event** instead of computing one point delta per event and fanning it out through a precomputed player-to-team index (§26-27) — this is the single most common reason a naive implementation falls over under real load.
- **Re-sorting a leaderboard table on every read** with an `ORDER BY ... LIMIT` query instead of maintaining an incrementally-sorted structure like a sorted set (§29-30).
- **Locking a contest reactively** based on a match-status update instead of proactively scheduling the lock against the fixed kickoff time, and separately re-guarding every join request at write time regardless of scheduler timing (§32-33).
- **Making payout a single, non-idempotent loop** with no per-entry durable "already paid" record — a crash partway through then either skips remaining winners on a naive restart-from-scratch-is-too-scary manual fix, or double-pays everyone on a naive blind restart (§36-38).
- **Comparing invite codes with a plain string `.equals()`** instead of a constant-time comparison, leaking timing information about a supposedly-secret code (§44).
- **Bolting fraud checks on after the fact** as a batch job that reviews completed entries, instead of evaluating the highest-confidence signals synchronously before an entry is allowed to proceed at all (§45-46).

---

# 52. Testing Strategy

- **Lineup validation tests** — construct lineups that violate the salary cap, violate role-count bounds, and violate the per-real-team cap, one violation at a time, and assert each is independently caught.
- **Contest-slot concurrency tests** — fire many simultaneous join requests against a contest with exactly one remaining slot and assert exactly one succeeds, regardless of thread interleaving.
- **Wallet concurrency tests** — fire many simultaneous debits against a balance that can satisfy at most one of them, and assert exactly one succeeds and the balance never goes negative.
- **Fan-out correctness tests** — inject a sequence of match events for one player and assert every team that drafted that player (and no team that didn't) reflects the correct cumulative point delta.
- **Leaderboard rank tests** — assert `topN` and `rankOf` remain consistent with each other after a burst of concurrent `incrementScore` calls, including exact ties resolved by the encoded tiebreaker (§29).
- **Lock-timing tests** — assert a join request submitted a few milliseconds after the scheduled kickoff is rejected even if the scheduled lock job itself has not yet run.
- **Payout idempotency tests** — run the payout job to completion, then run it again from scratch, and assert zero additional wallet credits occur on the second run.
- **Cancellation/refund tests** — cancel a contest mid-lifecycle and assert every entry fee is refunded exactly once, using the same idempotency mechanism as payout.

---

# 53. Suggested Future Enhancements

- **Multi-sport universal rule engine** — generalizing `SportRuleSet` beyond simple count ranges to arbitrary constraint expressions, for sports with more exotic composition rules (e.g., a maximum combined "credits" system with fractional player costs).
- **Dynamic, in-play pricing** — adjusting a player's draft price between contests based on recent real-world form, reusing the pluggable-pricing idea from this series' Vending Machine guide's promotional-pricing strategy, applied to player valuation instead of product pricing.
- **Tiered, dynamically-sized contests** — auto-scaling a contest's `totalSlots` based on real-time demand in the minutes before lock, which would require revisiting §20's atomic reservation to also handle a concurrently-changing ceiling, not just a concurrently-changing floor.
- **Cross-match leaderboards and season-long standings** — aggregating per-contest results into a longer-lived ranking, which introduces its own new idempotency question (crediting season points exactly once per completed contest).
- **Real-money regulatory geofencing** — restricting contest types or entry-fee tiers by jurisdiction, layered in as an additional pre-condition alongside §46's fraud gate.

---

# 54. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Model the contest lifecycle as an explicit state machine and explain why "open" and "locked" alone isn't enough. (§12-15)
2. Design lineup validation (salary cap, role composition) so a new sport never requires touching existing code. (§16-18)
3. 10,000 slots, 40,000 simultaneous join attempts at the deadline — design the fix, precisely. (§19-21)
4. The same user's wallet is debited from two devices at once — is this the same bug as question 3, or a different one? Design the fix. (§22-24)
5. One match event needs to reach millions of already-submitted teams' scores within seconds — design the pipeline, and explain why per-event full recomputation fails. (§25-27)
6. Design a leaderboard that stays correctly ranked for 20 million entries without re-sorting on every write. (§28-30)
7. A contest must lock at a scheduled kickoff time even under scheduler delay or clock drift — design the mechanism that makes this correct regardless of timing. (§32-33)
8. The payout job crashes partway through 20 million entries and is restarted from scratch — design payout so this is safe. (§36-38)
9. Extend the design to private, invite-only contests, and to basic multi-account fraud detection — what's genuinely new, and what's reused unchanged? (§43-46)

---

# 55. Final Takeaway

Nearly everything hard in this system reduces to the same recurring shape: **a check and an act that must happen as one indivisible operation, no matter how many concurrent requests are racing to perform it** — a contest slot (§20-21), a wallet balance (§23-24), a scheduled lock (§33), and a payout (§37-38) are four different resources guarded by the exact same atomic-conditional-update pattern this series has now applied to seats, cash, contest slots, and money in turn. The second recurring shape is just as important: **separate what changed from who is affected**, computing a small delta once and fanning it out through a precomputed index (§25-27), rather than re-deriving the full answer from scratch for every reader on every event. Recognizing which of these two shapes a new requirement actually is — and reaching for the atomic guard or the delta-plus-index pattern accordingly — is the transferable skill this guide is really teaching.

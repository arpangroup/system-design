# Design a Snake Game — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Player input (arrow keys) --buffered--> [ Direction Buffer ]
                                              |
                          every fixed tick    v
                                      [ Game Loop / Simulation ]
                                              |
                             move snake, check collisions, grow on food
                                              |
                          +-------------------+-------------------+
                          v                                       v
                  [ Renderer (canvas) ]                  [ Score / Leaderboard ]
                                                                    |
                                                          [ Contest / Prize Ledger ]
```

A Snake game looks like a beginner exercise, but a *system* built around it — one supporting local and real-time networked multiplayer, a pluggable AI opponent (from a simple pathfinder to a mathematically guaranteed-survival strategy to a trained learning agent), persistent scores, and prize-bearing high-score contests — is a genuinely rich real-time systems design exercise, distinct in almost every way from a turn-based game. This guide builds one from scratch: a fixed-timestep game loop decoupled from rendering, an efficient growing-body representation, input buffering that prevents the classic "reversed into yourself" bug, collision detection that doesn't scan the whole board, server-authoritative real-time multiplayer with client-side prediction, pathfinding and guaranteed-survival AI strategies, a learned agent (and why Snake's state space forces a genuinely different training approach than a small, fully-enumerable game would), and contest design with real-time-appropriate anti-cheat.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Explain why a real-time game needs a fixed-timestep simulation loop decoupled from rendering, rather than acting immediately on every input event.
- Represent a growing snake body so that movement and growth are both cheap, constant-time operations, not an array shift or a full re-scan.
- Correctly buffer player input to prevent the direction queued mid-tick from ever reversing the snake directly into its own body.
- Detect wall, self, and opponent collisions in constant time using a hash-set of currently-occupied cells, instead of scanning the board.
- Design real-time networked multiplayer with a server-authoritative tick simulation, and explain why client-side prediction and reconciliation are necessary for it to feel responsive.
- Design and compare three AI strategies — greedy pathfinding, a mathematically guaranteed-survival Hamiltonian cycle, and a trained learned agent — and explain precisely why Snake's state space forces function approximation where a small game like Tic-Tac-Toe did not.
- Apply SOLID principles and recognizable design patterns (Strategy, Observer, Facade) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design Snake" is a favorite real-time systems question precisely because, unlike a turn-based game, it forces a candidate to reason about *time itself* as a first-class design concern: a fixed simulation tick rate, the gap between when a server knows the truth and when a client sees it, and the very different collision/pathfinding computations a continuously-moving, growing body requires compared to a static board. It also sets up a natural, honest contrast with a turn-based game like Tic-Tac-Toe on the exact same three axes — networking (continuous ticks vs. discrete turns), AI (spatial pathfinding vs. exhaustive game-tree search), and training (huge state space requiring function approximation vs. a small table that can be learned exactly) — which is precisely the comparison this guide draws throughout. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Records model immutable per-tick snapshots precisely; `ArrayDeque` gives O(1) growth/shrink at both ends for the snake body |
| Runnable artifact | HTML5 Canvas + vanilla JavaScript | Single-file, dependency-free, playable directly in a browser — matches the guide's own worked examples exactly |
| Simulation loop | Fixed-timestep tick, decoupled from render rate | Deterministic, network-reproducible simulation independent of a client's actual frame rate |
| Collision detection | Hash set of occupied cells | O(1) collision check per tick, instead of scanning every body segment |
| AI pathfinding | Breadth-first search (shortest path) and a Hamiltonian cycle | BFS is the simplest correct shortest-path search on an unweighted grid; a Hamiltonian cycle is the standard technique for a provably-never-traps-itself Snake AI |
| Trained AI | Function-approximated Q-learning (small neural network) | Snake's state space is too large to enumerate exactly, unlike Tic-Tac-Toe's ~5,478 states — this is the guide's deliberate point of contrast |
| Networked multiplayer | WebSocket, server-authoritative fixed-tick simulation | The server's tick is the single source of truth; clients render a predicted, later-reconciled view |

---

# 5. Project Structure

```text
snake-game/
├── src/main/java/com/example/snake/
│   ├── domain/
│   │   └── Position.java, Direction.java, Snake.java, Food.java        // §10, §15-16
│   ├── loop/
│   │   └── GameLoop.java (fixed timestep)                              // §13
│   ├── input/
│   │   └── DirectionBuffer.java                                        // §19
│   ├── collision/
│   │   └── OccupiedCellIndex.java                                      // §22
│   ├── multiplayer/
│   │   ├── LocalMultiSnakeSession.java                                 // §25
│   │   ├── NetworkedTickSession.java (server-authoritative)             // §27
│   │   └── ClientPredictor.java (prediction + reconciliation)          // §29
│   ├── ai/
│   │   ├── SnakeAiStrategy.java (Strategy interface)                   // §31
│   │   ├── GreedyPathfindingStrategy.java (BFS)                        // §33
│   │   ├── HamiltonianCycleStrategy.java                                // §36
│   │   └── LearnedAgent.java, AgentTrainer.java (function approx.)     // §43
│   ├── score/
│   │   └── ScoreRepository.java, Leaderboard.java                      // §46
│   └── contest/
│       ├── HighScoreContest.java                                       // §48
│       └── TickAuditLog.java (anti-cheat)                               // §50
├── src/test/java/com/example/snake/
│   ├── CollisionDetectionTest.java
│   ├── HamiltonianCycleNeverDiesTest.java
│   └── ServerReconciliationTest.java
└── artifact/
    └── snake-arena.html   -- the runnable canvas game, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design Snake."*

Even for a game this familiar, the intentionally open prompt needs narrowing: single-player only, or local/networked multiplayer with several snakes on one board? Does "AI" mean a hardcoded pathfinder, a genuinely trained learning agent, or both? Is there a persistent leaderboard, and a prize-bearing high-score contest? Real-time games also raise a question turn-based ones don't: what tick rate, and how tolerant does the design need to be of network latency for multiplayer to still feel responsive? The answers reshape which parts of the design carry real weight, and asking them signals the difference between reciting "just move the array" and actually designing *this* system.

---

# 7. Functional Requirements

- **Move the snake continuously** in the currently-set direction at a fixed tick rate, growing by one segment each time it eats food.
- **Detect collisions**: with the walls (or wrap-around, if configured), with the snake's own body, and — in multiplayer — with any opponent snake's body.
- **Spawn food** at random unoccupied cells after each one is eaten.
- **Support local multiplayer**: multiple snakes on one shared board, each independently controlled.
- **Support real-time networked multiplayer** between remote players, with the game feeling responsive despite network latency.
- **Support an AI-controlled snake** at multiple strength levels, including a genuinely trained learning agent.
- **Track scores and a persistent leaderboard**, and support a prize-bearing high-score contest.

---

# 8. Non-Functional Requirements

- **Deterministic simulation**: given the same sequence of inputs, the game state must evolve identically every time — essential for both fair networked play and reproducible testing.
- **Low input-to-action latency**: a direction change should be reflected within at most one simulation tick, never several.
- **Real-time responsiveness under network latency**: a networked player's own snake must feel immediately responsive to their input, despite the round-trip delay to the authoritative server.
- **Correctness under concurrency**: in multiplayer, two snakes moving into the same cell in the same tick must be resolved by one well-defined rule, never a race.
- **Bounded resource usage for AI/training**: a learned agent's model size must stay bounded regardless of how long training runs, and training itself must not block the game's own simulation loop.
- **Fairness and integrity for prize contests**: the server, never a client, must be the sole source of truth for a contest-eligible score.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Position** — a single grid cell, `(x, y)`.
- **Direction** — one of `UP/DOWN/LEFT/RIGHT`, the snake's current heading.
- **Snake** — an ordered sequence of positions (head to tail), plus a current direction and a pending-growth counter.
- **Food** — a single position the snake grows by reaching.
- **Tick** — one discrete simulation step: apply the buffered direction, move, check collisions, check food, advance.
- **Game Session** — the full state of one game in progress: the board, all snakes, current food, and elapsed ticks — the unit both local and networked play operate on identically.

---

# 10. Identifying the Core Domain Entities

```java
public record Position(int x, int y) { }
public enum Direction {
    UP(0, -1), DOWN(0, 1), LEFT(-1, 0), RIGHT(1, 0);
    public final int dx, dy;
    Direction(int dx, int dy) { this.dx = dx; this.dy = dy; }
    public boolean isOpposite(Direction other) { return this.dx == -other.dx && this.dy == -other.dy; }
}

public class Snake {
    private final Deque<Position> body = new ArrayDeque<>(); // head at the front, tail at the back
    private Direction currentDirection;
    private int pendingGrowth = 0;

    public Position head() { return body.peekFirst(); }
    public boolean isAlive = true;
}

public record Food(Position position) { }

public class GameSession {
    private final int width, height;
    private final List<Snake> snakes;
    private Food currentFood;
    private long tickCount = 0;
}
```

Modeling `Direction` as an enum carrying its own `(dx, dy)` delta and an `isOpposite` check, rather than a bare string or int, is what makes §17-19's reverse-direction guard a single, obvious method call rather than a fragile set of coordinate comparisons scattered wherever direction changes are handled.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Player input ------>|   DirectionBuffer        |
(arrow keys)        +-----------+--------------+
                                |  (consumed once per tick)
                                v
                    +------------------------+       +------------------------+
                    |       GameLoop           |------>|   Renderer (canvas)     |
                    |  fixed-timestep tick     |      +------------------------+
                    +-----------+--------------+
                                |
                    +-----------+-------------+
                    v                         v
          +------------------+     +------------------------+
          |  OccupiedCellIndex |     |     GameSession          |
          |  (collision, O(1)) |     |  snakes, food, tickCount |
          +------------------+     +-----------+--------------+
                                                |
                                +---------------+---------------+
                                v                                 v
                    +------------------------+       +------------------------+
                    |    ScoreRepository       |       |   HighScoreContest      |
                    |    Leaderboard           |       |   PrizeLedger            |
                    +------------------------+       +------------------------+
```

`GameLoop` is the single place per-tick logic lives — it consumes the current buffered direction, advances every snake, checks collisions via `OccupiedCellIndex`, checks food, and updates `GameSession` — with `Renderer` only ever reading the resulting state to draw it, never influencing the simulation itself.

---

# 12. Follow-up Question 2 — "Why Can't You Just Move and Redraw Immediately on Every Keypress?"

> **Interviewer:** *"The simplest possible implementation moves the snake and redraws the instant an arrow key is pressed. What's wrong with that?"*

Because a keypress and a "how far the snake should have moved by now" are two entirely different clocks — tying movement directly to input events means the snake's speed becomes an accident of how fast a player mashes keys, and networked play becomes impossible to keep consistent across clients pressing keys at different real-world instants. The correct design decouples **when input is accepted** from **when the simulation actually advances**: input is only ever *buffered*, and a fixed-rate tick is what actually moves anything, checks collisions, and updates state — exactly the separation a real-time game (as opposed to a turn-based one like Tic-Tac-Toe) fundamentally requires.

---

# 13. The Game Loop: Fixed-Timestep Simulation

```text
Wall-clock time:  0ms    50ms   100ms   150ms   200ms   250ms  ...
Render calls:     |------|------|-------|----|-------|----------->  (varies with device/frame rate)
Simulation ticks: |-------------|--------------|---------------->  (FIXED rate, e.g. every 120ms,
                                                                      regardless of render frame rate)

Each tick:  1. consume the current buffered direction (§17-19)
            2. move every snake's head one cell in that direction
            3. check collisions (§20-22)
            4. check food; grow and respawn food if eaten
            5. advance tickCount
```

Rendering can happen as often as the display allows (and can even interpolate smoothly *between* two simulation ticks for visual polish) — but the simulation itself only ever advances at the fixed tick rate, which is what makes the game's actual behavior fully independent of frame rate, and, critically, makes it possible for a server and multiple clients to all compute the *identical* sequence of ticks from the *identical* sequence of buffered inputs (§26-29).

---

# 14. Follow-up Question 3 — "How Do You Represent the Snake's Body So Movement and Growth Are Cheap, Not Shifting an Array Every Tick?"

> **Interviewer:** *"A naive representation stores the body as an array and shifts every element forward each tick. What's wrong with that at scale, and what's the fix?"*

Shifting every element of an array on every single tick costs `O(body length)` per tick, even though moving a snake is conceptually a **constant-time** operation: a new head cell is added at the front, and (unless the snake just ate food) the tail cell is removed from the back — nothing in the middle ever actually needs to move. The fix is a data structure offering O(1) insertion and removal at *both* ends: a **double-ended queue**.

---

# 15. Efficient Snake Body Representation

```text
Body, head-to-tail:  [ (5,5) (4,5) (3,5) (2,5) ]   (moving RIGHT... wait, direction is LEFT here)

Move one tick (no food eaten):
  addFirst( newHead )   -> [ (6,5) (5,5) (4,5) (3,5) (2,5) ]
  removeLast()          -> [ (6,5) (5,5) (4,5) (3,5) ]        -- net: O(1), nothing in the middle touched

Move one tick (food eaten this tick):
  addFirst( newHead )   -> [ (6,5) (5,5) (4,5) (3,5) (2,5) ]
  (tail NOT removed -- this is exactly how the snake grows by one segment)
```

Using an `ArrayDeque<Position>` (or an equivalent structure with true O(1) operations at both ends — a plain `LinkedList` also qualifies, though with worse cache locality) means the cost of moving the snake stays constant regardless of how long it has grown, which matters increasingly as a game session runs longer and the snake's body length grows into the hundreds.

---

# 16. Implementing Movement and Growth

```java
public class Snake {
    private final Deque<Position> body = new ArrayDeque<>();
    private Direction currentDirection;
    private int pendingGrowth = 0;
    private boolean alive = true;

    public void tickMove() {
        Position head = body.peekFirst();
        Position newHead = new Position(head.x() + currentDirection.dx, head.y() + currentDirection.dy);
        body.addFirst(newHead);

        if (pendingGrowth > 0) {
            pendingGrowth--; // grow: skip removing the tail this tick
        } else {
            body.removeLast();
        }
    }

    public void grow(int segments) {
        pendingGrowth += segments; // deferred: actually applied on the NEXT tickMove(), not immediately
    }
}
```

`grow` deliberately doesn't append a segment immediately — it just increments a counter that `tickMove` consults on the *next* tick, which keeps growth expressed in terms of the exact same "add head, conditionally remove tail" operation every other tick already uses, rather than needing a separate code path for "a tick where the snake got longer."

---

# 17. Follow-up Question 4 — "Two Direction Keys Are Pressed Within One Tick. Which Applies, and How Do You Prevent Reversing Into Yourself?"

> **Interviewer:** *"A player moving RIGHT presses UP and then, in the same tick window, LEFT. If LEFT is applied directly, the snake instantly reverses into its own neck. How do you prevent that?"*

By never applying a direction change **immediately** on keypress — instead, each keypress only updates a small **buffer** holding the *next* direction to apply, and the buffer's own write is itself guarded: a new direction is only accepted into the buffer if it isn't the exact opposite of the snake's **currently active** direction (not the previous buffered one, a subtlety §19 makes precise). This is the single most common real bug in a from-scratch Snake implementation, and it exists precisely because a snake's own body occupies the cell directly behind its head — reversing into it is an instant, always-avoidable self-collision.

---

# 18. Input Buffering and the Reverse-Direction Bug

```text
WRONG (checks against the BUFFERED direction, not the ACTIVE one):
  Snake moving RIGHT. Tick N: player presses UP -- buffer = UP (accepted: UP != opposite of RIGHT)
                       (before tick N+1 processes) player presses LEFT
                       -- checked against buffer (UP): LEFT != opposite of UP, so ACCEPTED
                       -- but the snake is STILL, right now, actually moving RIGHT --
                          LEFT is the exact opposite of RIGHT -- this WILL reverse it into itself
                          once UP hasn't been applied yet on a fast-enough double key-press

CORRECT (checks against the ACTIVE direction the snake is CURRENTLY executing):
  Buffer stores at most ONE pending direction. A new keypress is accepted into the buffer only if
  it is not the opposite of the snake's CURRENT direction (the one actually being executed this tick),
  not the opposite of whatever is already sitting in the buffer.
```

The fix requires being precise about *which* direction a new keypress is validated against — the snake's currently-executing direction, which only changes once per tick, not the buffer's contents, which can otherwise be overwritten multiple times between two ticks by a fast typist.

---

# 19. Implementing the Input Buffer

```java
public class DirectionBuffer {
    private Direction pending;

    public void submit(Direction requested, Direction currentActiveDirection) {
        if (!requested.isOpposite(currentActiveDirection)) {
            pending = requested; // accepted -- will be applied at the START of the next tick
        }
        // if it WAS the opposite of the currently-active direction, silently ignored --
        // NOT queued, NOT applied later, exactly as if the keypress never happened
    }

    public Direction consumeOrKeep(Direction currentActiveDirection) {
        Direction result = (pending != null) ? pending : currentActiveDirection;
        pending = null;
        return result;
    }
}
```

`consumeOrKeep` is called exactly once, at the start of every tick, and its result becomes the snake's new *currently-active* direction for that entire tick — which is precisely the value every subsequent `submit` call during that tick will be validated against, closing the loop correctly.

---

# 20. Follow-up Question 5 — "How Do You Detect Collision Efficiently Every Tick Without Scanning the Whole Board?"

> **Interviewer:** *"A naive collision check loops over every existing body segment to see if the new head matches one. What's wrong with that as the snake grows, and what's the fix?"*

Looping over every segment costs `O(body length)` per tick, and this cost is paid **every single tick**, compounding as the snake (and, in multiplayer, every opponent snake sharing the board) grows longer — at a long enough game session, this becomes a genuinely measurable per-tick cost. The fix is to maintain a **hash set of every currently-occupied cell**, updated incrementally exactly alongside the body's own `Deque` mutations (§16) — checking whether a new head position collides becomes a single O(1) set lookup, independent of how long any snake has grown.

---

# 21. Efficient Collision Detection via a Hash Set of Occupied Cells

```text
Naive:  for each snake: for each segment in that snake's body: if segment == newHead: COLLISION
        -- O(total segments across ALL snakes) per tick, paid EVERY tick

This guide's approach:  maintain Set<Position> occupiedCells, updated INCREMENTALLY:
        - when a segment is added (addFirst): occupiedCells.add(newHead)
        - when a segment is removed (removeLast): occupiedCells.remove(oldTail)
        Checking a new head for collision: occupiedCells.contains(newHead) -- O(1), ALWAYS
```

The set must be updated in exact lockstep with every `Deque` mutation from §16 — adding a head without registering it in the set, or removing a tail without unregistering it, would silently desynchronize the two structures, reintroducing exactly the kind of stale-state bug this guide's own house style treats as a first-class thing to design against, not merely test for after the fact.

---

# 22. Implementing Collision Detection

```java
public class OccupiedCellIndex {
    private final Map<Position, Snake> occupiedBy = new HashMap<>(); // WHICH snake owns each cell -- needed for multiplayer

    public void onSegmentAdded(Position position, Snake owner) { occupiedBy.put(position, owner); }
    public void onSegmentRemoved(Position position) { occupiedBy.remove(position); }

    public Optional<CollisionResult> checkCollision(Position newHead, int boardWidth, int boardHeight) {
        if (newHead.x() < 0 || newHead.x() >= boardWidth || newHead.y() < 0 || newHead.y() >= boardHeight) {
            return Optional.of(CollisionResult.WALL);
        }
        Snake occupant = occupiedBy.get(newHead);
        if (occupant != null) {
            return Optional.of(CollisionResult.BODY); // self OR opponent -- the caller distinguishes using `occupant`
        }
        return Optional.empty(); // no collision -- the move is safe
    }
}

public enum CollisionResult { WALL, BODY }
```

Mapping each occupied `Position` to the specific `Snake` that owns it (rather than a plain `Set<Position>`) is what lets multiplayer distinguish "collided with my own tail" from "collided with an opponent" — both are structurally a `BODY` collision, but which snake dies (and, in some rule variants, whether the *other* snake also takes damage) depends on knowing whose segment was actually hit.

---

# 23. Class Diagram: The Core Game Loop

```text
+------------------------+       +------------------------+
|       GameLoop            |------>|     GameSession          |
|  tick()                   |       |  snakes, food, tickCount |
+-----------+--------------+       +-----------+--------------+
            |                                    |
            v                                    v
+------------------------+       +------------------------+
|    DirectionBuffer        |       |         Snake             |
|  submit(), consumeOrKeep()|       |  body: Deque<Position>   |
+------------------------+       |  tickMove(), grow()      |
                                    +-----------+--------------+
                                                  |
                                                  v
                                    +------------------------+
                                    |   OccupiedCellIndex       |
                                    |  checkCollision(newHead)  |
                                    +------------------------+
```

`GameLoop.tick()` is the orchestrator that ties every mechanism introduced so far together in the correct order: consume the buffered direction (§19), move each snake (§16), check collisions via the index (§22), check for food, and advance — with `Renderer` (§11) only ever reading the resulting `GameSession` afterward, never participating in this sequence itself.

---

# 24. Follow-up Question 6 — "How Do You Support Local Multiplayer — Two Snakes, One Board — Cleanly?"

> **Interviewer:** *"Two players share one keyboard, one screen. How does the design extend from one snake to several without becoming a tangle of duplicated logic?"*

`GameSession` already models `snakes` as a **list**, not a single field (§10) — local multiplayer is simply that list holding more than one entry, each with its own `DirectionBuffer` fed by a different set of keys (e.g., arrow keys for one player, WASD for the other), and `OccupiedCellIndex` (§22) already correctly attributes every cell to whichever specific snake owns it, which is exactly what makes an opponent collision distinguishable from a self-collision with zero additional mechanism.

---

# 25. Modeling Multiple Snakes on One Shared Board

```text
GameSession.snakes = [ snakeA (player, arrow keys), snakeB (player, WASD), snakeC (AI, §31) ]

Each tick, GameLoop:
  for each snake in snakes (in a FIXED, deterministic order -- matters for tie-breaking, §26):
    if snake.isAlive: consume its OWN DirectionBuffer, move it, check its collision
  -- one snake dying does NOT stop the loop from processing the others this same tick
```

Processing snakes in a fixed, deterministic order (rather than, say, an unordered collection whose iteration order could vary) is what makes two-snakes-collide-head-on scenarios resolvable by a single, well-defined rule rather than depending on incidental iteration order — directly setting up §26's networked-multiplayer requirement that the exact same tick, given the exact same inputs, always produces the exact same outcome.

---

# 26. Follow-up Question 7 — "How Do You Support Real-Time Networked Multiplayer, Not Turn-Based Like Tic-Tac-Toe?"

> **Interviewer:** *"Tic-Tac-Toe's networked design relayed one move at a time, whenever a player happened to act. Snake moves continuously, every tick, for every player, simultaneously. How does that change the networking design?"*

Fundamentally: instead of relaying discrete moves, the server runs the **entire authoritative simulation itself**, at the fixed tick rate, continuously — each client only ever sends its *currently-buffered direction* (updated whenever the player presses a key), and the server folds every connected client's latest known direction into its own tick, then broadcasts the resulting `GameSession` snapshot to everyone. No client ever computes the "official" outcome of a tick; every client only ever *renders* what the server already decided.

---

# 27. Networked Multiplayer: Server-Authoritative Tick Simulation

```text
Client A: key press -> sends {direction: UP} to server (NOT "I moved to (5,5)" -- the server decides that)
Client B: key press -> sends {direction: LEFT} to server

Server, every fixed tick (independent of when either message arrived, as long as it's before the tick):
  1. GameLoop.tick() consumes whatever direction each client's DirectionBuffer currently holds
  2. Moves BOTH snakes, checks collisions, checks food (§13, §16, §22) -- IDENTICAL logic to local play
  3. Broadcasts the resulting GameSession snapshot to BOTH clients

Neither client's own simulation is ever treated as authoritative -- only the server's ticks are.
```

This is the exact same architectural principle the Tic-Tac-Toe guide's §22-23 already established (the server, never a client, is the sole source of truth) — applied here to a *continuous* simulation instead of discrete per-move validation, because Snake's real-time nature means there is no natural "one move, one validation" boundary to hang a simpler check on.

---

# 28. Follow-up Question 8 — "Network Latency Means a Client Could Feel Behind the Server. How Do You Keep It Feeling Responsive?"

> **Interviewer:** *"If a client only ever renders what the server tells it, and that requires a round trip, doesn't the player's own snake feel laggy, moving only after a network delay?"*

Yes — waiting for a server round-trip before showing *your own* movement would feel noticeably sluggish, which is precisely why real-time multiplayer games layer **client-side prediction** on top of the server-authoritative design: the client immediately simulates its own snake's movement locally, the instant a key is pressed, rendering that predicted result *before* the server's confirmation arrives — and later reconciles its prediction against the server's actual authoritative tick once it arrives, correcting silently if they disagree.

---

# 29. Client-Side Prediction and Server Reconciliation

```text
Client, locally, the instant a key is pressed:
  1. Apply the move to a LOCAL COPY of the snake immediately -- render it right away, no waiting
  2. Remember this predicted state, tagged with the tick number it predicted

Server's authoritative snapshot for that same tick number arrives (after network delay):
  3. Compare the server's actual state for that tick against the client's earlier PREDICTION
  4. If they MATCH: nothing visible happens -- the prediction was correct, already rendered
  5. If they DIFFER (e.g., a collision the client didn't predict because it didn't know an
     opponent's move yet): SNAP the client's rendered state to the server's authoritative one --
     a visible correction, but a rare one, and only when the prediction was genuinely wrong
```

The key property that makes this whole approach viable is that the *vast majority* of predictions turn out correct (a player's own movement rarely depends on information only the server had), so corrections are rare, brief, and far less disruptive than *always* waiting a full round-trip before ever showing movement at all — this is the standard technique behind why fast-paced multiplayer games feel responsive despite genuine network latency.

---

# 30. Follow-up Question 9 — "How Do You Plug In an AI-Controlled Snake?"

> **Interviewer:** *"From the game loop's point of view, should there be any difference between a human's buffered direction and an AI's?"*

None — exactly the same answer the Tic-Tac-Toe guide gave for its own AI integration (§24 there): an AI-controlled snake is simply something that computes a `Direction` given the current `GameSession`, submitted into that snake's own `DirectionBuffer` through the identical `submit()` call a human's keypress would use. `GameLoop.tick()` never needs to know or care which snakes are human-controlled and which are AI-controlled.

---

# 31. Designing the AI as a Pluggable Strategy

```java
public interface SnakeAiStrategy {
    Direction decideMove(GameSession session, Snake self);
}
```

Every AI difficulty level — from a simple pathfinder to a mathematically guaranteed-survival strategy to a trained learned agent — implements this identical interface, exactly mirroring the Tic-Tac-Toe guide's `AiStrategy` (§25 there): swapping strategies is a configuration change, never a change to `GameLoop` or either transport layer.

---

# 32. Follow-up Question 10 — "What's the Simplest AI, and What's Wrong With It?"

> **Interviewer:** *"Implement the simplest reasonable AI opponent for Snake. What's its obvious weakness?"*

The simplest *reasonable* strategy computes the shortest path to the food and follows it — reasonable, because a truly naive "move randomly" AI for Snake dies almost immediately and isn't an interesting comparison point the way it was for Tic-Tac-Toe. But shortest-path-to-food has a real, well-known weakness: it optimizes purely for "reach the food fastest," with no regard for whether doing so **traps the snake's own body** against a wall with no remaining escape route once it's grown long enough — a greedy strategy can walk itself into a dead end that was entirely avoidable.

---

# 33. Strategy 1: Greedy Shortest-Path (BFS)

```java
public class GreedyPathfindingStrategy implements SnakeAiStrategy {
    @Override
    public Direction decideMove(GameSession session, Snake self) {
        List<Position> path = breadthFirstSearch(session, self.head(), session.currentFood().position());
        if (path.isEmpty() || path.size() < 2) {
            return anySafeFallbackDirection(session, self); // no path found -- avoid immediate death, §37-38
        }
        Position nextStep = path.get(1); // path.get(0) is the current head itself
        return directionBetween(self.head(), nextStep);
    }

    private List<Position> breadthFirstSearch(GameSession session, Position start, Position goal) {
        Queue<Position> frontier = new ArrayDeque<>();
        Map<Position, Position> cameFrom = new HashMap<>();
        frontier.add(start);
        cameFrom.put(start, null);

        while (!frontier.isEmpty()) {
            Position current = frontier.poll();
            if (current.equals(goal)) return reconstructPath(cameFrom, goal);
            for (Position neighbor : walkableNeighbors(session, current)) {
                if (!cameFrom.containsKey(neighbor)) {
                    cameFrom.put(neighbor, current);
                    frontier.add(neighbor);
                }
            }
        }
        return List.of(); // no path exists to the food at all right now
    }
}
```

**Breadth-first search** is the correct, simplest algorithm here specifically because every edge on the grid (moving to any adjacent cell) has identical cost — on an unweighted graph, BFS is guaranteed to find the *shortest* path, with none of Dijkstra's or A*'s added complexity being necessary (A* would only earn its keep if some cells cost more to enter than others, which a plain Snake board doesn't have).

---

# 34. Follow-up Question 11 — "Greedy BFS Can Trap Itself in a Dead End. How Do You Build an AI That (Almost) Never Dies?"

> **Interviewer:** *"You've shown me greedy pathfinding can walk itself into a corner. Design an AI that's structurally safe from ever doing that."*

By abandoning "shortest path to food" as the goal entirely, and instead having the snake **always follow one fixed, precomputed cycle that visits every single cell on the board exactly once before returning to its start** — a **Hamiltonian cycle**. As long as the snake always moves along this cycle, it can *never* collide with its own body (the cycle, by definition, never crosses itself), which makes death from self-collision structurally impossible regardless of how long the snake grows — the strongest safety guarantee any of these strategies can offer.

---

# 35. Strategy 2: Hamiltonian Cycle — Guaranteed Survival

```text
A Hamiltonian cycle on a grid: a single closed loop touching every cell exactly once.

Example, on a 4x4 grid (one valid cycle, arrows show the fixed order the snake always follows):
  (0,0)->(1,0)->(2,0)->(3,0)
    ^                     |
    |                     v
  (0,1)<-(1,1)<-(2,1)<-(3,1)
    |
    v
  (0,2)->(1,2)->(2,2)->(3,2)
    ^                     |
    |                     v
  (0,3)<-(1,3)<-(2,3)<-(3,3)

The snake ALWAYS moves to the NEXT cell in this fixed sequence, regardless of where the food
currently is -- it will eventually reach every cell, including wherever food happens to be,
guaranteed, simply by continuing to follow the cycle.
```

The guarantee here is unconditional: because the snake's own body only ever occupies a *contiguous stretch* of this single cycle at any moment, and the cycle never revisits a cell out of order, the snake's head can never run into its own body — this is a structural property of the cycle itself, not something that needs to be checked or verified at runtime.

---

# 36. Implementing a Hamiltonian Cycle Strategy

```java
public class HamiltonianCycleStrategy implements SnakeAiStrategy {
    private final Map<Position, Position> nextInCycle; // precomputed ONCE, at startup, for the board's fixed size

    public HamiltonianCycleStrategy(int width, int height) {
        this.nextInCycle = precomputeCycle(width, height); // a fixed, deterministic cycle covering every cell
    }

    @Override
    public Direction decideMove(GameSession session, Snake self) {
        Position next = nextInCycle.get(self.head());
        return directionBetween(self.head(), next);
    }
}
```

The cycle is computed **once**, at startup, for the board's fixed dimensions — not recomputed per tick — since the cycle itself never changes; the only per-tick work is a single map lookup, making this strategy not just safe but also computationally the *cheapest* of the three, despite being conceptually the most elaborate to set up.

---

# 37. Follow-up Question 12 — "If a Hamiltonian Cycle Is Guaranteed-Safe, Why Not Always Use It?"

> **Interviewer:** *"You've just described a strategy that can never die from self-collision. Why would anyone ever use the greedy pathfinder instead?"*

Because "never dies" and "plays well" are different goals, and the Hamiltonian cycle strategy is honestly quite bad at the second one: it visits cells in a fixed order **regardless of where the food actually is**, meaning it can take an extremely long, meandering path to reach food sitting just a few cells away in Manhattan distance but far away along the cycle's fixed route — a human or a greedy pathfinder would reach that same food far faster. The Hamiltonian cycle strategy optimizes purely for survival; greedy BFS optimizes purely for speed; neither is strictly better, they're different points on the same tradeoff curve.

---

# 38. The Efficiency vs. Safety Tradeoff

```text
                    Speed to food         Death risk          Computational cost
Greedy BFS          Fast (optimal path)   Real (can self-trap) Cheap (one BFS per tick)
Hamiltonian Cycle   Slow (fixed route)    None (structural)     Cheapest (one lookup per tick)

A practical HYBRID: follow the Hamiltonian cycle by default, but take a greedy shortcut
whenever it's PROVABLY safe to do so (e.g., the shortcut rejoins the cycle at a point the
snake's body has already passed, so no future segment of the cycle gets permanently skipped) --
this is a real, more sophisticated strategy used in competitive Snake AI, combining both
strategies' strengths at the cost of a more involved safety proof per shortcut taken.
```

Naming this tradeoff explicitly — rather than presenting the Hamiltonian cycle as an unconditionally "better" strategy — is the correct way to answer this follow-up: a good design offers *both* strategies as genuinely different difficulty/behavior tiers, exactly as the Tic-Tac-Toe guide's own §37-38 did for minimax versus a trained agent, rather than treating one as simply an upgrade of the other.

---

# 39. Follow-up Question 13 — "The Prompt Says 'Train' an AI. Snake's State Space Is Far Bigger Than Tic-Tac-Toe's. What Changes?"

> **Interviewer:** *"Tic-Tac-Toe had about 5,478 reachable states, small enough for an exact lookup table. What's the equivalent number for Snake, and does the same approach still work?"*

It does not, and by an enormous margin: a Snake state includes the exact positions of every body segment, the food's position, and the current direction — on even a modest 20x20 board, the number of distinct reachable configurations is astronomically larger than Tic-Tac-Toe's ~5,478, far too many to ever store in an exact table, let alone visit each one enough times during training to learn a reliable value for it. This is precisely the point where reinforcement learning must abandon an exact table and adopt **function approximation** — typically a small neural network — that *generalizes* from the states it has actually seen to the vastly larger number it hasn't.

---

# 40. Why Snake Needs Function Approximation, Not an Exact Table

```text
Tic-Tac-Toe:  ~5,478 reachable states -- EXACT table: one row per state, learned independently,
              zero generalization needed or possible between unrelated states.

Snake:        astronomically many reachable states (grows combinatorially with board size and
              snake length) -- an exact table could never be fully populated, let alone learned;
              a NEURAL NETWORK instead learns a compact set of weights that GENERALIZES: two
              states that look similar (food slightly to the left vs. slightly further left)
              naturally produce similar predicted values, without needing to have visited
              either exact state during training.
```

This is the central, honest technical distinction this guide draws against its own Tic-Tac-Toe companion: the *algorithm* (Q-learning, reward-driven self-play) is conceptually the same, but the *representation* of the learned policy must change entirely once the state space stops being small enough to enumerate — a lesson that generalizes directly to any real-world reinforcement learning problem too large for an exact table, which is the overwhelming majority of them.

---

# 41. Designing the State Representation and Reward Function

```text
State representation (a small, fixed-size vector, NOT the raw board):
  - Danger straight-ahead, danger to the left, danger to the right (3 booleans -- immediate collision risk)
  - Current direction (4 booleans, one-hot)
  - Food direction relative to head: is it up/down/left/right of the head (4 booleans)
  -- a compact ~11-value vector, not a full board snapshot -- this is what makes the network
     small and what lets it GENERALIZE across the huge underlying state space (§40)

Reward function:
  +10   for eating food
  -10   for dying (wall, self, or opponent collision)
  -0.01 per tick survived with no other event (a small, constant penalty that encourages
        reaching food EFFICIENTLY, rather than the agent learning to survive by endlessly
        stalling without ever approaching the food at all)
```

The reward function's small per-tick penalty is a deliberately chosen detail, not an afterthought — without it, an agent that has learned "never dying" is a valid strategy could simply oscillate forever in a safe pocket of the board, technically maximizing survival time while never actually playing the game the reward was meant to teach it to play.

---

# 42. Follow-up Question 14 — "Walk Through the Actual Training Loop for a Function-Approximated Agent"

> **Interviewer:** *"Concretely, how does training actually proceed, tick by tick?"*

The agent plays repeated episodes (each ending in death), and after every tick, it uses the observed transition — the state before the move, the action taken, the reward received, and the resulting state — to nudge its network's weights so that its **predicted** value for that action moves closer to the reward actually received plus a discounted estimate of the best value available from the resulting state, exactly the same Q-learning update rule the Tic-Tac-Toe guide used (§35-36 there), just applied to network weights via gradient descent instead of directly overwriting a table cell.

---

# 43. Implementing a Minimal Trainable Agent

```java
public class LearnedAgent implements SnakeAiStrategy {
    private final SmallNeuralNetwork network; // a few dense layers, input = state vector (§41), output = 4 Q-values

    @Override
    public Direction decideMove(GameSession session, Snake self) {
        double[] state = encodeState(session, self);
        double[] qValues = network.predict(state); // one value per possible direction
        return Direction.values()[argMax(qValues)];
    }
}

public class AgentTrainer {
    private static final double LEARNING_RATE = 0.001;
    private static final double DISCOUNT = 0.95;
    private double epsilon = 0.3; // epsilon-greedy exploration, exactly as in the Tic-Tac-Toe guide's §35-36

    public void trainOneStep(LearnedAgent agent, double[] stateBefore, int actionTaken,
                              double reward, double[] stateAfter, boolean episodeEnded) {
        double[] qBefore = agent.network.predict(stateBefore);
        double target;
        if (episodeEnded) {
            target = reward;
        } else {
            double[] qAfter = agent.network.predict(stateAfter);
            target = reward + DISCOUNT * max(qAfter); // standard Q-learning bootstrap
        }
        double[] targetVector = qBefore.clone();
        targetVector[actionTaken] = target;
        agent.network.trainStep(stateBefore, targetVector, LEARNING_RATE); // one gradient-descent update
    }
}
```

`trainOneStep` is called after every single tick during training (not just at episode end), which is what lets the agent start improving its predictions well before any one episode finishes — a meaningful contrast with the Tic-Tac-Toe agent's own per-move updates (§36 there), since here the network's *weights* shift a small amount on each update rather than a table *cell* being overwritten directly, and it's this gradual, generalized weight-shifting that lets the agent transfer what it learned in one state to the many similar, never-directly-visited states nearby.

---

# 44. Class Diagram: AI Strategies

```text
+------------------------+
|    SnakeAiStrategy        |
|    <<interface>>          |
|  decideMove(session,self) |
+-----------+--------------+
      ^      ^      ^
+----------------+ +------------------+ +------------------------+
| Greedy          | | Hamiltonian     | | LearnedAgent            |
| Pathfinding     | | CycleStrategy   | | (network, trained via   |
| Strategy (BFS)  | |                 | |  AgentTrainer, §43)     |
+----------------+ +------------------+ +------------------------+
```

All three strategies slot into the identical `SnakeAiStrategy` interface introduced in §31 — adding a fourth (e.g., the hybrid strategy sketched in §38) means implementing one interface, never modifying `GameLoop`, either transport layer, or any of the other strategies.

---

# 45. Follow-up Question 15 — "How Do You Track Scores Across Many Games and Many Players?"

> **Interviewer:** *"A single session's score is easy. How do you turn that into persistent, per-player records and a leaderboard?"*

Every completed game session — ended by every snake dying, or a configured session time limit — should emit a single, immutable **result event** (player identity, final score, ticks survived) that a `ScoreRepository` durably records and folds into each player's running best-score and history, exactly the same "compute once, persist immediately" discipline this series applies consistently, and identical in spirit to the Tic-Tac-Toe guide's own §40-42.

---

# 46. Score Management and Leaderboards

```java
public class ScoreRepository {
    private final Map<String, PlayerRecord> recordsByPlayerId = new ConcurrentHashMap<>();

    public void recordResult(String playerId, int finalScore, long ticksSurvived) {
        PlayerRecord record = recordsByPlayerId.computeIfAbsent(playerId, id -> new PlayerRecord(id));
        record.gamesPlayed++;
        record.bestScore = Math.max(record.bestScore, finalScore);
        record.totalScore += finalScore;
    }

    public List<PlayerRecord> leaderboard(int topN) {
        return recordsByPlayerId.values().stream()
            .sorted(Comparator.comparingInt((PlayerRecord r) -> r.bestScore).reversed())
            .limit(topN)
            .toList();
    }
}
```

Ranking the leaderboard by each player's **best** score, not their most recent one, is a deliberate design choice appropriate for a game where score naturally varies run to run — it rewards a player's peak performance rather than punishing an unlucky recent attempt, which is the conventional (and generally preferred) leaderboard semantics for a high-score-style game.

---

# 47. Follow-up Question 16 — "How Do You Support a Contest With a Prize, for a Game That Isn't Head-to-Head Like Tic-Tac-Toe's Match Series?"

> **Interviewer:** *"Snake isn't naturally 'best of 5' the way Tic-Tac-Toe was. What does a prize-bearing contest look like for a high-score game instead?"*

A **high-score contest**: every entrant plays within a fixed window (a time limit, or a fixed number of attempts), each attempt's score is recorded exactly as any ordinary game would be, and whoever posts the single highest verified score once the window closes wins the prize — structurally simpler than the Tic-Tac-Toe guide's best-of-series aggregation (§43-44 there), since there's no need to track a running series score, only each entrant's best individual result.

---

# 48. Designing a High-Score Contest and Prize Distribution

```java
public class HighScoreContest {
    private final Instant closesAt;
    private final BigDecimal prizePool;
    private final Map<String, Integer> bestScoreByPlayer = new ConcurrentHashMap<>();
    private boolean prizeDistributed = false;

    public void recordAttempt(String playerId, int score) {
        if (Instant.now().isAfter(closesAt)) return; // attempts after closing don't count, even if submitted
        bestScoreByPlayer.merge(playerId, score, Math::max);
    }

    public void distributePrizeIfClosed(PrizeLedger ledger) {
        if (prizeDistributed || Instant.now().isBefore(closesAt)) return;
        String winner = bestScoreByPlayer.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow(() -> new IllegalStateException("Contest closed with no attempts recorded"));
        ledger.credit(winner, prizePool); // idempotent by construction: prizeDistributed guards a single call
        prizeDistributed = true;
    }
}
```

The `Instant.now().isAfter(closesAt)` check inside `recordAttempt` is what makes the contest window a genuinely enforced boundary rather than an advisory one — a score submitted a moment after closing is simply never recorded, which matters because §49-50's anti-cheat discussion depends on there being one crisp, server-enforced instant that unambiguously separates "counts" from "doesn't."

---

# 49. Follow-up Question 17 — "How Do You Prevent Cheating in a Real-Time Prize Contest, Differently Than in Tic-Tac-Toe?"

> **Interviewer:** *"Tic-Tac-Toe validated discrete moves. Snake's score accumulates continuously, tick by tick, over a much longer session. What's different about anti-cheat here?"*

The core principle is identical (§26-27: the server, never a client, is the sole source of truth) — but the *surface area* for tampering is larger, because a real-time session has vastly more individual ticks than a Tic-Tac-Toe game has moves, and a client could, in principle, claim an inflated final score without the server having genuinely simulated every tick that supposedly produced it. The fix is that the server's own tick-by-tick simulation (§27) is the *only* source a contest score is ever read from — a client never gets to report "my final score was X" at all; the server already knows the true score because it computed every tick itself.

---

# 50. Contest Integrity: Server-Simulated Ticks, Not Client-Reported Scores

```text
UNTRUSTED (never do this for a real-money contest):
  Client plays a full session locally --> client sends "my final score was 340" to the server
  --> server credits the contest entry -- a modified client can simply LIE about the number

TRUSTED (this guide's design):
  Every tick's outcome is computed by the SERVER's own GameLoop (§27), from directions the
  client only ever SUGGESTS -- the score displayed to a client is always a REFLECTION of a
  number the server itself derived, never a value the client is trusted to report back
```

This is the same lesson as the Tic-Tac-Toe guide's own §46-47, generalized: whatever a client reports about itself is data, never fact, the instant real value depends on the outcome — the only change here is that "the outcome" accumulates continuously over many ticks instead of being decided by a single terminal move, which is exactly why the server must run the *entire* simulation, not merely validate discrete submitted claims.

---

# 51. Capacity Estimation: Concurrent Real-Time Sessions and Tick Bandwidth

```text
Assume: 20,000 concurrent networked sessions, each with 2 players, ticking at 8 ticks/sec

Total ticks/sec across the platform = 20,000 * 8                    = 160,000 ticks/sec
Per-tick broadcast payload (snake positions, food, score; ~150 bytes, both players) ≈ 150 bytes
Total outbound bandwidth = 160,000 * 150 bytes                       ≈ 24 MB/sec platform-wide

This is a MEANINGFULLY larger continuous bandwidth commitment than Tic-Tac-Toe's equivalent
estimate (§48 there) -- a direct consequence of Snake being a CONTINUOUS real-time simulation
broadcasting every tick, rather than a turn-based game only transmitting on the rare occasions
a player actually acts. This is the concrete cost of real-time responsiveness (§28-29).
```

At this scale, tick bandwidth (not simulation CPU cost, which remains cheap per §21-22's O(1) collision checks) is the actual bottleneck resource to provision for — a direct, quantifiable consequence of choosing a continuously-ticking real-time design over a discrete, event-driven turn-based one.

---

# 52. Full Worked Example: One Networked Tick, Traced End to End

```text
1. Client A (playing "Alice") presses the UP arrow key
     a. Locally: DirectionBuffer.submit(UP, currentActiveDirection) -- accepted, not opposite (§19)
     b. Client-side prediction immediately renders Alice's snake moving UP, before any server reply (§29)
     c. The client sends {direction: UP} to the server over the existing WebSocket connection
2. Server's next scheduled tick fires (independent of exactly when step 1c's message arrived, as
   long as it arrived before this tick, §27):
     a. GameLoop.tick() consumes Alice's DirectionBuffer (now holding UP) and Bob's (unchanged)
     b. Both snakes move; OccupiedCellIndex.checkCollision() checked for each (§21-22) -- no collision
     c. Food not reached this tick -- no growth, no respawn
     d. tickCount advances; the resulting GameSession snapshot is broadcast to both clients
3. Client A receives the server's snapshot for this tick number:
     a. Compares it against its own earlier PREDICTION for the same tick -- they MATCH (§29)
     b. No visible correction needed -- the already-rendered predicted frame was correct
4. Several ticks later, Alice eats food:
     a. Snake.grow(1) called (§16); ScoreRepository eventually records the finished session's
        score once the game ends (§46), reading only from the server's own tally, never a
        client-reported number (§49-50)
```

Every mechanism this guide introduced via a follow-up question appears somewhere in this one tick's (and its immediate aftermath's) trace — the fixed tick loop, input buffering, prediction/reconciliation, and server-authoritative scoring are not independent, optional features, they are the actual steps a single real networked tick passes through in this design.

---

# 53. Final Architecture Diagram

```text
Client A <--WebSocket--> [ NetworkedTickSession ] <--WebSocket--> Client B
 (predicts locally,               |
  reconciles on snapshot,  +------v------+
  §29)                     |  GameLoop    |
                            |  tick()      |
                            +------+------+
                                    |
                    +---------------+---------------+
                    v                                 v
        +------------------+              +------------------------+
        | OccupiedCellIndex  |              |     GameSession          |
        +------------------+              |  snakes, food, tickCount |
                                            +-----------+--------------+
                                                        |
                                    +-------------------+-------------------+
                                    v                                       v
                        +------------------------+              +------------------------+
                        |    ScoreRepository       |              |   HighScoreContest      |
                        |    Leaderboard           |              |   PrizeLedger            |
                        +------------------------+              +------------------------+

(Local/offline mode, structurally identical GameLoop, no network layer:)
Keyboard input --> [ DirectionBuffer per snake ] --> [ GameLoop ] --> Renderer + ScoreRepository
                                                            ^
                                                            |
                            [ SnakeAiStrategy: Greedy BFS | Hamiltonian Cycle | LearnedAgent ]
```

---

# 54. Design Patterns Used Throughout This Guide

- **Strategy** — `SnakeAiStrategy` (§31-44: greedy BFS, Hamiltonian cycle, learned agent) is a single swappable interface behind which every AI difficulty tier lives, interchangeable via configuration alone.
- **Facade** — `GameLoop` hides body movement, collision detection, and food handling behind a single `tick()` entry point every transport layer and every AI adapter depends on.
- **Observer** (implicit) — the renderer and the score repository each react to a finished tick's resulting `GameSession` without the loop itself needing to know how many downstream consumers exist or what they do with it.
- **Adapter** — an AI's `decideMove` result is submitted through the exact same `DirectionBuffer.submit()` a human keypress uses, letting `GameLoop` treat both uniformly (§30-31).
- **Repository** — `ScoreRepository` and `PrizeLedger` each encapsulate their own persistence and aggregation concerns behind a narrow, purpose-specific interface.

---

# 55. SOLID Principles Applied

- **Single Responsibility** — `Snake` only tracks body/movement/growth; `OccupiedCellIndex` only tracks collision state; `DirectionBuffer` only mediates input timing — none of the three knows how to do the others' job, even though `GameLoop` composes all three on every tick.
- **Open/Closed** — adding a new AI strategy (or a new contest format) means implementing one interface, never modifying `GameLoop`, either transport layer, or `HighScoreContest`.
- **Liskov Substitution** — every `SnakeAiStrategy` implementation must honestly return a legal `Direction` from `decideMove`, so the game loop can call any of them interchangeably without special-casing a particular strategy.
- **Interface Segregation** — `SnakeAiStrategy` exposes exactly one method, so a trivial pathfinder isn't forced to depend on concepts (network weights, training hyperparameters) only the learned agent actually needs.
- **Dependency Inversion** — both local and networked sessions depend only on `GameLoop`'s public contract, never reimplementing simulation rules themselves, so the exact same rules engine serves both transports without duplication.

---

# 56. Common Mistakes When Building This Yourself

```text
MISTAKE                                                 CORRECT APPROACH (this guide's section)
Moving the snake directly on keypress, not a fixed tick   Fixed-timestep loop decoupled from input (§12-13)
Shifting an array to move/grow the snake body              O(1) Deque operations at both ends (§14-16)
Validating a new direction against the BUFFERED direction  Validate against the CURRENTLY ACTIVE one (§17-19)
Scanning every segment to check collision every tick        Incrementally-updated occupied-cell set (§20-22)
Trusting a client's own simulation in networked play         Server-authoritative ticks (§26-27)
Waiting for a server round-trip before rendering own moves  Client-side prediction + reconciliation (§28-29)
Using an exact Q-table for Snake, copying Tic-Tac-Toe's fix Function approximation is required at this scale (§39-40)
Crediting a contest prize based on a client-reported score  Server-derived score only, ever (§49-50)
```

---

# 57. Testing Strategy

- **Collision detection tests** — verify a wall collision, a self-collision, and (in multiplayer) an opponent collision are each correctly detected, and that a legal move into empty space is correctly not flagged.
- **Input-buffer tests** — assert that a direction opposite to the *currently active* one is always rejected, and that rapid, repeated direction submissions within one tick only ever leave the *last valid* one pending.
- **Hamiltonian-cycle-never-dies tests** — run the strategy for many thousands of ticks on a fixed board size and assert the snake never records a self-collision, confirming the structural guarantee actually holds in the implementation, not just on paper.
- **Server-reconciliation tests** — simulate a client prediction that deliberately diverges from a scripted server outcome, and assert the client correctly snaps to the server's authoritative state rather than silently diverging further.
- **Contest integrity tests** — submit an attempt after a contest's `closesAt` instant and assert it is never recorded, and attempt to distribute a prize twice for the same contest, asserting the second call is a no-op.

---

# 58. Suggested Future Enhancements

- **Wrap-around board variant** — treating the board edges as connected (exiting the right edge re-enters on the left) rather than a wall collision, changing §21-22's collision rule for the boundary case specifically.
- **Power-ups** (temporary speed boost, temporary invincibility) — new transient entities layered on top of the existing tick loop, following the same "computed once per tick, read by the renderer" pattern already used for food.
- **Spectator mode** for networked matches, broadcasting the authoritative `GameSession` snapshot to read-only observers in addition to the active players, mirroring the Tic-Tac-Toe guide's own identical suggestion (§55 there).
- **Larger, deeper neural networks with replay buffers** — a genuine step up from §41-43's minimal trainable agent, storing past transitions and sampling from them during training (experience replay) for more stable, sample-efficient learning.
- **Regional matchmaking and multiple contest tiers** — grouping players by latency region before matching them for networked play, and running simultaneous contests at different entry-fee/prize tiers.

---

# 59. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Why does a real-time game need a fixed-timestep simulation loop decoupled from rendering, unlike a turn-based game? (§12-13)
2. Represent the snake's growing body so movement and growth are both O(1), and explain precisely why a plain array isn't. (§14-16)
3. Design input handling so a fast double-keypress can never reverse the snake directly into its own body. (§17-19)
4. Design real-time networked multiplayer, including why the server — not either client — must run the actual simulation. (§26-27)
5. A networked client would feel laggy if it only ever rendered server-confirmed state. Design the fix, and explain what happens when it's wrong. (§28-29)
6. Design an AI that is structurally guaranteed to never die from self-collision, and explain its real weakness compared to a greedy pathfinder. (§34-38)
7. Tic-Tac-Toe's state space was small enough for an exact Q-table. Why doesn't that approach work for Snake, and what replaces it? (§39-41)
8. Design a prize-bearing contest for a high-score game, and explain specifically how you prevent a client from reporting a fabricated score. (§47-50)

---

# 60. Final Takeaway

Every hard decision in this guide traces back to one recurring idea, made sharper by contrast with this series' own turn-based Tic-Tac-Toe guide: **a continuous, real-time system needs an explicit, decoupled notion of time wherever a turn-based one could get away with reacting directly to events** — a fixed tick governs simulation independent of render rate or input timing (§12-13); a server's own continuous tick stream, not discrete relayed moves, is what makes real-time multiplayer trustworthy (§26-27); prediction and reconciliation exist specifically to hide network latency from a *continuous* experience in a way a turn-based game's players simply never notice (§28-29). Recognizing when a design problem is fundamentally about discrete events versus continuous time is what determines which half of this guide's toolkit actually applies.

---

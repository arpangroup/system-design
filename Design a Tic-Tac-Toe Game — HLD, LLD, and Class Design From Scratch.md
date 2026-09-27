# Design a Tic-Tac-Toe Game — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Player X / Player O (human or AI) --move--> [ Game Engine ] --validates, applies--> Board
                                                    |
                                             [ Win/Draw Checker ]
                                                    |
                                        [ Score Repository ] <--updates--> Leaderboard
                                                    |
                                        [ Match/Contest Manager ] --prize awarded at series end-->
```

Tic-Tac-Toe looks trivial, but a *system* built around it — one supporting local pass-and-play, real-time networked multiplayer, multiple AI opponents (from a random-move novice to a mathematically unbeatable player), a trainable learning agent, persistent score tracking, and prize-bearing contests — is a genuinely rich object-oriented and distributed-systems design exercise. This guide builds one from scratch: an efficient board representation with cheap win detection, a clean separation between the game engine and its transport layer (local vs. networked), a pluggable AI strategy hierarchy (random, heuristic, minimax, and a self-trained reinforcement-learning agent), score/leaderboard management, and match/contest design with prize distribution and anti-cheat considerations.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Represent a Tic-Tac-Toe board so that checking for a win is a cheap, constant-time operation rather than a repeated nested-loop scan.
- Model game state as an explicit state machine, making illegal moves (playing out of turn, playing an occupied cell, playing after the game has ended) structurally rejected rather than merely checked after the fact.
- Separate the game engine's pure rules from its transport layer, so the identical engine serves both local pass-and-play and real-time networked multiplayer.
- Design a pluggable AI strategy hierarchy — random, rule-based heuristic, minimax with alpha-beta pruning, and a self-trained Q-learning agent — behind one common interface.
- Explain precisely what "training" means for a solved game like Tic-Tac-Toe, and design a real self-play training loop (state representation, reward shaping, exploration vs. exploitation).
- Design score tracking, leaderboards, and prize-bearing match/contest structures, including the server-authoritative validation a real-money contest requires.
- Apply SOLID principles and recognizable design patterns (Strategy, State, Factory, Observer) to keep the system extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design Tic-Tac-Toe" is a deceptively rich interview question specifically because the game's rules are trivial enough that a candidate has no excuse to hide weak design behind incidental complexity — every design choice (state modeling, strategy pluggability, network/local separation) is fully visible and fully judged on its own merits. Extending it with AI opponents, training, and prize contests turns a toy exercise into a genuine tour through classic design patterns, a real (if small-scale) reinforcement-learning problem, and real distributed-systems concerns (turn authority, anti-cheat) that show up in every larger multiplayer system too. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language (design) | Java 21 | Sealed interfaces and records model board/state/move precisely; exhaustive `switch` keeps state transitions compiler-checked |
| Runnable artifact | HTML5 Canvas + vanilla JavaScript | Single-file, dependency-free, playable directly in a browser — matches the guide's own worked examples exactly |
| Win detection | Bitmask representation | O(1) win check via a precomputed mask table, instead of repeated nested-loop scans |
| Minimax AI | Minimax with alpha-beta pruning | The mathematically correct way to build a provably unbeatable player for a small, fully-observable game like this |
| Trainable AI | Tabular Q-learning via self-play | Tic-Tac-Toe's state space (~5,478 legal, reachable states) is small enough for an exact table, making the RL concepts fully visible without needing function approximation |
| Networked multiplayer | WebSocket, server-authoritative | Real-time turn exchange with the server as the single source of truth for move legality (§22-23) |

---

# 5. Project Structure

```text
tic-tac-toe/
├── src/main/java/com/example/tictactoe/
│   ├── domain/
│   │   └── Board.java, Move.java, Player.java, GameResult.java        // §10, §14
│   ├── engine/
│   │   ├── GameState.java (State pattern)                              // §16-17
│   │   └── GameEngine.java                                             // §17, §20
│   ├── transport/
│   │   ├── LocalGameSession.java                                       // §20
│   │   └── NetworkedGameSession.java (WebSocket)                       // §21, §23
│   ├── ai/
│   │   ├── AiStrategy.java (Strategy interface)                        // §25
│   │   ├── RandomStrategy.java                                         // §27
│   │   ├── HeuristicStrategy.java                                      // §29
│   │   ├── MinimaxStrategy.java                                        // §32
│   │   └── QLearningAgent.java, QLearningTrainer.java                  // §36
│   ├── score/
│   │   └── ScoreRepository.java, Leaderboard.java                      // §42
│   └── contest/
│       ├── MatchSeries.java, Contest.java                              // §44
│       └── ContestValidator.java (anti-cheat)                          // §47
├── src/test/java/com/example/tictactoe/
│   ├── WinDetectionTest.java
│   ├── MinimaxUnbeatableTest.java
│   └── QLearningConvergenceTest.java
└── artifact/
    └── tic-tac-toe.html   -- the runnable canvas game, §1's worked design made playable
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design Tic-Tac-Toe."*

Even for a game this simple, the intentionally open prompt needs narrowing: is this purely local pass-and-play, or does it need real-time networked multiplayer? How many AI difficulty levels, and does "AI" mean a hardcoded algorithm, a genuinely trained learning agent, or both? Does score tracking persist across sessions, and is there a leaderboard? Does "contest with a winning prize" imply real money (with all the integrity/anti-cheat obligations that brings) or just a virtual badge/points? The answers reshape which parts of the design carry real weight — a request specifically naming multiplayer, AI training, and prize contests (as this one does) is signaling that the interesting design surface is well beyond the game's trivial rules.

---

# 7. Functional Requirements

- **Play a standard 3x3 Tic-Tac-Toe game** between two players (human or AI), enforcing turn order and legal moves.
- **Detect a win, loss, or draw** immediately upon the move that produces it.
- **Support local pass-and-play** on a single device, and **networked multiplayer** between two remote players.
- **Support multiple AI opponents** at different strength levels, selectable by the human player.
- **Support training an AI agent** via self-play, with observable training progress.
- **Track scores** (wins/losses/draws) per player, persisted across sessions, with a leaderboard.
- **Support a match/contest mode**: a best-of-N series between two players (or a bracket among many), culminating in a prize awarded to the overall winner.

---

# 8. Non-Functional Requirements

- **Correctness of game rules**: an illegal move (wrong turn, occupied cell, move after game end) must be structurally rejected, never silently accepted.
- **Low latency for local play and AI moves**: a human player should never perceive a noticeable delay waiting for an AI's move, even at the highest (minimax) difficulty.
- **Real-time responsiveness for networked play**: a move made by one player should be reflected on the opponent's screen within a fraction of a second under normal network conditions.
- **Fairness and integrity for prize contests**: the server, not either client, must be the single source of truth for which moves are legal and who won — a client can never be trusted to self-report a win in a contest with a real prize.
- **Extensibility**: adding a new AI strategy or a new contest format should require adding new code, not modifying the existing game engine or transport layer.
- **Bounded state for the learning agent**: the trainable AI's internal representation must not grow unboundedly — Tic-Tac-Toe's small, fully-enumerable state space makes this achievable exactly, not just approximately.

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Board** — the 3x3 grid of cells, each empty or occupied by X/O.
- **Move** — a single placement: which player, which cell, at what point in the game's sequence.
- **Game State** — the game's current phase (in progress, X won, O won, draw) — an explicit, closed set of values, not an inferred condition re-derived on every check.
- **Player** — a participant, human or AI, identified by a symbol (X or O) and, for scoring purposes, a persistent identity.
- **AI Strategy** — the pluggable decision-making behind an AI player's move selection.
- **Match Series / Contest** — a sequence of individual games between (or among) players, aggregated into a result that determines a prize winner.

---

# 10. Identifying the Core Domain Entities

```java
public enum Symbol { X, O }
public enum GameResult { IN_PROGRESS, X_WINS, O_WINS, DRAW }

public record Move(Symbol player, int cellIndex, int moveNumber) { } // cellIndex: 0-8

public class Board {
    private final Symbol[] cells = new Symbol[9]; // null = empty
    private int moveCount = 0;

    public boolean isEmpty(int cellIndex) { return cells[cellIndex] == null; }
    public void place(int cellIndex, Symbol player) { cells[cellIndex] = player; moveCount++; }
    public Symbol at(int cellIndex) { return cells[cellIndex]; }
    public boolean isFull() { return moveCount == 9; }
}

public record Player(String playerId, String displayName, Symbol symbol, boolean isAi) { }
```

Encoding `GameResult` as its own enum, rather than inferring "is the game over" from repeatedly re-scanning the board on every query, is the same discipline this series has applied to spot/ticket lifecycles and rate-limit decisions elsewhere: compute the fact once, at the moment it becomes true, and carry it forward as an explicit value rather than re-derived state.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Human/AI move --->|      GameEngine          |
                    |  validate, apply, check |
                    +-----------+--------------+
                          |               |
                +---------+       +---------+
                v                          v
      +------------------+     +------------------------+
      | LocalGameSession   |     | NetworkedGameSession    |
      | (same-device        |     | (WebSocket, server-      |
      |  pass-and-play)     |     |  authoritative, §21-23) |
      +------------------+     +------------------------+
                                                |
                          +---------------------+---------------------+
                          v                                           v
              +------------------------+                +------------------------+
              |    ScoreRepository       |                |   MatchSeries/Contest    |
              |    Leaderboard           |                |   Prize distribution      |
              +------------------------+                +------------------------+
```

`GameEngine` is deliberately the single place game rules live — both `LocalGameSession` and `NetworkedGameSession` call the exact same engine to validate and apply a move, differing only in *where the move came from* and *how the result is communicated back*, which is precisely what §19-20's follow-up question is about.

---

# 12. Follow-up Question 2 — "How Do You Represent the Board So Win-Checking Is Cheap, Not a Nested-Loop Scan Every Move?"

> **Interviewer:** *"A naive win-check loops over all 8 possible winning lines (3 rows, 3 columns, 2 diagonals), checking 3 cells each, after every single move. What's a cheaper approach?"*

Represent each player's occupied cells as a 9-bit **bitmask** (one bit per cell), and precompute all 8 winning-line bitmasks once, at startup. Checking for a win after a move becomes a single bitwise AND between the player's current mask and each winning-line mask, compared for equality — a handful of cheap integer operations, replacing repeated cell-by-cell comparisons, and trivially fast even though Tic-Tac-Toe's scale never actually demands this optimization for correctness (it's included here because the *technique* — bitmask state, precomputed pattern table — generalizes to many larger board-game and combinatorial-search problems where it genuinely matters).

---

# 13. Efficient Board Representation and Win Detection

```text
Cell indices:        Winning line bitmasks (bit i set = cell i is part of this line):
   0 | 1 | 2          Rows:    0b000000111, 0b000111000, 0b111000000
  -----------         Columns: 0b001001001, 0b010010010, 0b100100100
   3 | 4 | 5          Diagonals: 0b100010001, 0b001010100
  -----------
   6 | 7 | 8

Player X occupies cells {0, 4, 8} (a diagonal)  ->  xMask = 0b100010001
Check: (xMask & diagonalMask) == diagonalMask   ->  TRUE -- X has won
```

Maintaining one integer bitmask **per player** (rather than a single combined board representation requiring 2 bits per cell to distinguish X/O/empty) keeps the win check itself a single-operand comparison — no need to first extract which player occupies a given line's cells before comparing.

---

# 14. Implementing the Board and Win Checker

```java
public class Board {
    private static final int[] WINNING_MASKS = {
        0b000000111, 0b000111000, 0b111000000,  // rows
        0b001001001, 0b010010010, 0b100100100,  // columns
        0b100010001, 0b001010100                 // diagonals
    };

    private int xMask = 0;
    private int oMask = 0;
    private int moveCount = 0;

    public void place(int cellIndex, Symbol player) {
        int bit = 1 << cellIndex;
        if (player == Symbol.X) xMask |= bit; else oMask |= bit;
        moveCount++;
    }

    public boolean hasWon(Symbol player) {
        int mask = (player == Symbol.X) ? xMask : oMask;
        for (int winMask : WINNING_MASKS) {
            if ((mask & winMask) == winMask) return true; // O(1) per line, 8 lines total -- constant work
        }
        return false;
    }

    public boolean isFull() { return moveCount == 9; }
    public boolean isOccupied(int cellIndex) { return ((xMask | oMask) & (1 << cellIndex)) != 0; }
}
```

`isOccupied` reuses the same bitmask representation for move legality checking — `xMask | oMask` cheaply gives "every occupied cell, regardless of which player," which is exactly what §16-17's move-validation logic needs before ever attempting to place a piece.

---

# 15. Follow-up Question 3 — "How Do You Model Turn-Taking and Prevent an Illegal Move?"

> **Interviewer:** *"What stops a player from moving twice in a row, playing on an occupied cell, or playing after the game has already ended?"*

By making the game's current phase an explicit **state machine**, exactly the same technique this series' Parking Lot guide used for spot lifecycles (§29-30 there) — a `GameState` object encodes precisely which operations are legal from the current phase, so calling "make a move" while the state is already `X_WINS` or `DRAW` is a structurally rejected operation, not a bug waiting for a missing `if` check somewhere in the engine.

---

# 16. Modeling Game State as a State Machine

```text
IN_PROGRESS --(a move completes a line for the mover)--> {X_WINS | O_WINS}
IN_PROGRESS --(a move fills the board with no winner)--> DRAW
IN_PROGRESS --(a legal move, game continues)--> IN_PROGRESS (still, just the turn advances)

Illegal, structurally prevented:
  X_WINS.applyMove(...)   -- the game has already concluded, no further moves are legal
  IN_PROGRESS.applyMove(move from the WRONG player) -- turn order violated
  IN_PROGRESS.applyMove(move on an ALREADY-OCCUPIED cell) -- cell legality violated
```

Every terminal state (`X_WINS`, `O_WINS`, `DRAW`) simply doesn't expose a legal further transition — precisely the same structural-impossibility principle the Parking Lot guide's `SpotState` used, applied here to a game's turn sequence instead of a physical spot's occupancy.

---

# 17. Implementing Game State Transitions

```java
public interface GameState {
    GameState applyMove(Board board, Move move); // returns the NEW state, or throws if illegal
}

public class InProgressState implements GameState {
    @Override
    public GameState applyMove(Board board, Move move) {
        if (board.isOccupied(move.cellIndex())) {
            throw new IllegalMoveException("Cell " + move.cellIndex() + " is already occupied");
        }
        board.place(move.cellIndex(), move.player());
        if (board.hasWon(move.player())) {
            return move.player() == Symbol.X ? new XWinsState() : new OWinsState();
        }
        if (board.isFull()) {
            return new DrawState();
        }
        return this; // game continues, still IN_PROGRESS
    }
}

public class XWinsState implements GameState {
    @Override
    public GameState applyMove(Board board, Move move) {
        throw new IllegalStateException("Game has already ended: X wins");
    }
}

public class GameEngine {
    private GameState currentState = new InProgressState();
    private final Board board = new Board();

    public void makeMove(Move move) {
        currentState = currentState.applyMove(board, move); // delegates the TRANSITION to the state itself
    }
}
```

`GameEngine` itself contains no `if (gameOver) { throw ... }` check anywhere — it simply delegates every move to whichever `GameState` it currently holds, and that state object either performs the transition or throws, exactly mirroring the delegation structure the Parking Lot guide's `ParkingSpot.markOccupied()` used for its own state machine.

---

# 18. Class Diagram: The Core Game Engine

```text
+------------------------+        +------------------------+
|       GameEngine         |------->|        Board             |
|  makeMove(move)          |       |  place(), hasWon(),      |
|  currentState: GameState |       |  isOccupied(), isFull()  |
+-----------+--------------+        +------------------------+
            |
            v
+------------------------+
|       GameState           |
|    <<interface>>          |
|  applyMove(board, move)   |
+-----------+--------------+
      ^      ^      ^      ^
+----------+ +--------+ +--------+ +--------+
|InProgress| | XWins  | | OWins  | | Draw   |
|State     | | State  | | State  | | State  |
+----------+ +--------+ +--------+ +--------+
```

This diagram is deliberately small — the entire rules engine for Tic-Tac-Toe fits in two classes and a four-state machine — which is exactly the point of using this game as a teaching vehicle: the design *patterns* on display (State, and Strategy in §25 onward) are fully visible without being obscured by incidental domain complexity.

---

# 19. Follow-up Question 4 — "How Do You Support Local Pass-and-Play AND Networked Multiplayer Without Duplicating Game Logic?"

> **Interviewer:** *"The rules are identical either way — only WHERE a move comes from, and HOW the result gets communicated, differ. How do you avoid writing the win/turn logic twice?"*

By keeping `GameEngine` (§17-18) completely ignorant of *where* a `Move` originated or *who* needs to be told about the result — it only ever validates and applies moves against its own state machine. A thin **transport layer** sits on top: `LocalGameSession` reads both players' moves from the same device's input and re-renders the same local board; `NetworkedGameSession` receives a move from one client, applies it to the *same* `GameEngine`, and relays the result to the other client over the network — neither transport implementation contains a single line of game-rule logic itself.

---

# 20. Separating Game Engine from Transport: Local vs. Networked Play

```text
LocalGameSession:
  Player 1 clicks a cell --> Move --> GameEngine.makeMove(move) --> re-render board on the SAME screen
  Player 2 clicks a cell --> Move --> GameEngine.makeMove(move) --> re-render board on the SAME screen

NetworkedGameSession:
  Player 1's client sends Move over WebSocket --> SERVER's GameEngine.makeMove(move)
                                                        |
                                        broadcast the resulting board state to BOTH clients
  Player 2's client sends Move over WebSocket --> SERVER's GameEngine.makeMove(move)
                                                        |
                                        broadcast the resulting board state to BOTH clients
```

The critical architectural difference is *where* `GameEngine` actually runs: in `LocalGameSession`, it can safely run on the player's own device since there's no other party to defraud; in `NetworkedGameSession`, it **must** run on the server (§22-23), because a client-side engine could be tampered with to report a false result — a distinction that becomes existential once real prizes are on the line (§43-47).

---

# 21. Implementing a Networked Multiplayer Session

```java
public class NetworkedGameSession {
    private final GameEngine engine; // runs SERVER-SIDE, authoritative
    private final WebSocketConnection playerXConnection;
    private final WebSocketConnection playerOConnection;

    public void onMoveReceived(WebSocketConnection sender, Move move) {
        Symbol expectedSymbol = (sender == playerXConnection) ? Symbol.X : Symbol.O;
        if (move.player() != expectedSymbol) {
            sender.sendError("You cannot submit a move for the other player's symbol");
            return;
        }
        try {
            engine.makeMove(move); // the SAME GameEngine class as LocalGameSession -- zero duplicated rules
            broadcastBoardState();
        } catch (IllegalMoveException | IllegalStateException e) {
            sender.sendError(e.getMessage());
        }
    }

    private void broadcastBoardState() {
        BoardSnapshot snapshot = engine.currentSnapshot();
        playerXConnection.send(snapshot);
        playerOConnection.send(snapshot);
    }
}
```

Notice `NetworkedGameSession` imports and calls the *exact same* `GameEngine` class `LocalGameSession` would — the only genuinely new logic here is connection/identity bookkeeping (which socket is X, which is O) and broadcasting, which is precisely the intended payoff of §19's separation.

---

# 22. Follow-up Question 5 — "Two Networked Moves Arrive at Nearly the Same Time. How Do You Prevent Both Being Applied?"

> **Interviewer:** *"Network jitter means two moves — one from each player — could arrive at the server within milliseconds of each other. What stops both from being applied, out of turn order?"*

`GameEngine`'s own state machine already structurally prevents this — `InProgressState.applyMove` (§17) rejects a move from the wrong player before it ever touches the board, so even if both moves arrive nearly simultaneously, whichever one the server's single-threaded (or per-session-locked) processing handles first transitions the engine's internal turn expectation, and the second move is rejected by the *same* validation logic already in place, not by any new mechanism specifically built for network timing.

---

# 23. Turn Authority and Move Validation on the Server

```text
Server processes moves for a SINGLE session strictly sequentially (either via a single-threaded
event loop per session, or a lock scoped to that one session) -- this is a SMALL-SCALE echo of
the Rate Limiter guide's atomic check-and-decrement requirement (§35 there): the "read current
state, validate, apply" sequence for one game session must not be split across two overlapping
operations, or the exact same class of race condition reappears.

Move from Player O arrives while it's actually X's turn (network reordering, or a buggy/
malicious client) --> InProgressState.applyMove() rejects it via the SAME turn-order check
already built into §17's state machine -- no separate "networked-only" validation path needed.
```

This is a small but important insight: the server doesn't need a *separate* anti-cheat-specific validation layer for turn order — the game engine's own correctness (already required for local play) is exactly the mechanism that also defeats a malicious or buggy networked client, as long as the server, not either client, is the one actually running it.

---

# 24. Follow-up Question 6 — "How Do You Plug In an AI Opponent Without the Game Engine Caring Whether It's Playing a Human or an AI?"

> **Interviewer:** *"From the engine's point of view, should there be any difference between a human's move and an AI's move?"*

None at all, and that's the correct design: an AI opponent is simply something that produces a `Move` given the current `Board` — exactly the same shape as a human clicking a cell and the client packaging that click into a `Move`. `GameEngine.makeMove(move)` never needs an `if (isAiTurn)` branch anywhere; a thin adapter asks the AI strategy for its move and submits it through the identical code path a human's move would take.

---

# 25. Designing the AI as a Pluggable Strategy

```java
public interface AiStrategy {
    int selectMove(Board board, Symbol aiSymbol); // returns a cellIndex, 0-8
}

public class AiPlayer {
    private final AiStrategy strategy;
    private final Symbol symbol;

    public Move decideMove(Board board, int moveNumber) {
        int cellIndex = strategy.selectMove(board, symbol);
        return new Move(symbol, cellIndex, moveNumber);
    }
}
```

Every AI difficulty level — from a novice random-mover to an unbeatable minimax solver to a self-trained learning agent — implements this identical `AiStrategy` interface, which is the textbook **Strategy** pattern applied directly: swapping difficulty levels is a configuration change, never a change to `AiPlayer`, `GameEngine`, or either transport session.

---

# 26. Follow-up Question 7 — "What's the Simplest AI, and What's Wrong With It?"

> **Interviewer:** *"Implement the simplest possible AI opponent. What's its obvious weakness?"*

The simplest correct `AiStrategy` implementation picks uniformly at random among the currently-empty cells — trivial to implement, and genuinely useful as a true "easy" difficulty for a beginner or a child, but it has no concept of blocking an imminent opponent win or taking an imminent win of its own, meaning it will happily lose games it could have easily won or blocked, purely by chance.

---

# 27. Strategy 1: Random Move Selection

```java
public class RandomStrategy implements AiStrategy {
    private final Random random = new Random();

    @Override
    public int selectMove(Board board, Symbol aiSymbol) {
        List<Integer> emptyCells = board.emptyCellIndices();
        return emptyCells.get(random.nextInt(emptyCells.size()));
    }
}
```

This is deliberately the *entire* implementation — no board evaluation, no lookahead — which is precisely why it's the correct choice for an "Easy" difficulty tier: a genuinely weak opponent, not an artificially handicapped strong one.

---

# 28. Follow-up Question 8 — "How Do You Make an AI That at Least Blocks Obvious Losses?"

> **Interviewer:** *"A slightly better AI should at least take an immediate winning move if one exists, and block the opponent's immediate winning move otherwise. Design that."*

A small, fixed **priority-ordered rule list**, checked in order on every move: (1) if the AI has an immediate winning move available, take it; (2) otherwise, if the opponent has an immediate winning move available, block it; (3) otherwise, fall back to a simple positional preference (center, then corners, then edges — the classic Tic-Tac-Toe heuristic ordering). This produces a noticeably stronger, still-beatable "Medium" difficulty without the computational cost of a full lookahead search.

---

# 29. Strategy 2: Heuristic/Rule-Based AI

```java
public class HeuristicStrategy implements AiStrategy {
    private static final int[] POSITIONAL_PRIORITY = {4, 0, 2, 6, 8, 1, 3, 5, 7}; // center, corners, edges

    @Override
    public int selectMove(Board board, Symbol aiSymbol) {
        Optional<Integer> winningMove = findWinningMove(board, aiSymbol);
        if (winningMove.isPresent()) return winningMove.get();

        Symbol opponent = (aiSymbol == Symbol.X) ? Symbol.O : Symbol.X;
        Optional<Integer> blockingMove = findWinningMove(board, opponent);
        if (blockingMove.isPresent()) return blockingMove.get();

        for (int cell : POSITIONAL_PRIORITY) {
            if (board.isEmpty(cell)) return cell;
        }
        throw new IllegalStateException("No empty cell found on a non-full board");
    }

    private Optional<Integer> findWinningMove(Board board, Symbol player) {
        for (int cell : board.emptyCellIndices()) {
            Board hypothetical = board.copy();
            hypothetical.place(cell, player);
            if (hypothetical.hasWon(player)) return Optional.of(cell);
        }
        return Optional.empty();
    }
}
```

`findWinningMove` is reused for **both** the "can I win" check and the "must I block" check, simply called with a different `player` argument — a small but instructive example of not duplicating logic just because it's being used for two conceptually different purposes (offense vs. defense) within the same strategy.

---

# 30. Follow-up Question 9 — "How Do You Build an AI That Literally Cannot Lose?"

> **Interviewer:** *"Design a 'Hard' AI that is provably unbeatable — not just strong, but mathematically guaranteed to never lose."*

By exhaustively exploring the game tree: **minimax** recursively simulates every possible sequence of remaining moves from the current position, assuming both players play optimally (the AI maximizes its own outcome, the opponent minimizes it — hence the name), and picks the move leading to the best guaranteed outcome. Because Tic-Tac-Toe's entire game tree is small enough to fully explore (at most 9! ≈ 362,880 move sequences, far fewer in practice once illegal continuations are pruned), minimax doesn't need to approximate anything — it computes the mathematically optimal move with certainty, which is precisely why an AI built this way can be proven to never lose (it will win or draw every game, depending on how well the opponent plays).

---

# 31. Strategy 3: Minimax with Alpha-Beta Pruning

```text
minimax(board, depth, isMaximizing):
  if board.hasWon(AI_SYMBOL): return +10 - depth   -- prefer WINNING SOONER (higher score for shallower wins)
  if board.hasWon(OPPONENT_SYMBOL): return -10 + depth -- prefer LOSING LATER, if a loss is somehow forced
  if board.isFull(): return 0                       -- a draw

  if isMaximizing (AI's turn):
    best = -infinity
    for each empty cell: simulate AI's move, best = max(best, minimax(..., depth+1, false))
    return best
  else (opponent's turn):
    best = +infinity
    for each empty cell: simulate opponent's move, best = min(best, minimax(..., depth+1, true))
    return best
```

Subtracting/adding `depth` from the terminal scores is the detail that makes minimax prefer winning **as quickly as possible** and, if a loss is somehow unavoidable, delaying it as long as possible — without this adjustment, minimax would still play optimally in terms of win/loss/draw outcome, but could pick an unnecessarily slow path to an already-guaranteed win, which looks strange (and occasionally exploitable against a human opponent hoping for a mistake) even though it's technically still correct.

---

# 32. Implementing Minimax

```java
public class MinimaxStrategy implements AiStrategy {

    @Override
    public int selectMove(Board board, Symbol aiSymbol) {
        int bestScore = Integer.MIN_VALUE;
        int bestCell = -1;
        for (int cell : board.emptyCellIndices()) {
            Board hypothetical = board.copy();
            hypothetical.place(cell, aiSymbol);
            int score = minimax(hypothetical, 1, false, aiSymbol, Integer.MIN_VALUE, Integer.MAX_VALUE);
            if (score > bestScore) { bestScore = score; bestCell = cell; }
        }
        return bestCell;
    }

    private int minimax(Board board, int depth, boolean isMaximizing, Symbol aiSymbol, int alpha, int beta) {
        Symbol opponent = (aiSymbol == Symbol.X) ? Symbol.O : Symbol.X;
        if (board.hasWon(aiSymbol)) return 10 - depth;
        if (board.hasWon(opponent)) return depth - 10;
        if (board.isFull()) return 0;

        if (isMaximizing) {
            int best = Integer.MIN_VALUE;
            for (int cell : board.emptyCellIndices()) {
                Board next = board.copy();
                next.place(cell, aiSymbol);
                best = Math.max(best, minimax(next, depth + 1, false, aiSymbol, alpha, beta));
                alpha = Math.max(alpha, best);
                if (beta <= alpha) break; // PRUNE -- the opponent already has a better option elsewhere, stop exploring
            }
            return best;
        } else {
            int best = Integer.MAX_VALUE;
            for (int cell : board.emptyCellIndices()) {
                Board next = board.copy();
                next.place(cell, opponent);
                best = Math.min(best, minimax(next, depth + 1, true, aiSymbol, alpha, beta));
                beta = Math.min(beta, best);
                if (beta <= alpha) break; // PRUNE, symmetric to the maximizing branch above
            }
            return best;
        }
    }
}
```

Alpha-beta pruning (the `if (beta <= alpha) break` lines) doesn't change minimax's outcome at all — the move it selects is provably identical to unpruned minimax — it only skips exploring branches that a rational opponent would never actually allow to be reached, cutting the number of positions evaluated dramatically without sacrificing any correctness.

---

# 33. Follow-up Question 10 — "The Prompt Says 'Train' an AI Player. Minimax Doesn't Need Training — What Does 'Training' Actually Mean Here?"

> **Interviewer:** *"You've built a perfect, unbeatable AI with zero training. So what is there left to 'train'? Isn't this already solved?"*

Minimax is a **solver** — it computes the correct move by exhaustive search, encoding no accumulated experience at all; ask it the same question twice and it re-derives the identical answer from scratch every time. **Training** means something categorically different: building an agent that **improves its move selection through repeated experience** (playing many games, observing outcomes, adjusting its future behavior accordingly) rather than through exhaustive real-time search — the correct framing for "training" here is **reinforcement learning**, where an agent learns a policy from self-play rather than being handed one by an algorithm.

---

# 34. Reinforcement Learning: Framing Tic-Tac-Toe as a Learning Problem

```text
State:   the current board configuration (which cells are X, O, or empty) -- ~5,478 legal,
         reachable states exist for Tic-Tac-Toe (a small enough number to store EXACTLY,
         in a table, rather than needing a neural network or other function approximator)
Action:  which empty cell to play next
Reward:  +1 for a win, -1 for a loss, 0 for a draw or a non-terminal move
Policy:  a mapping from (state, action) -> an estimated VALUE of taking that action in that
         state -- the agent's own learned "opinion" about how good each possible move is
```

Framing the problem this way is what makes Tic-Tac-Toe an ideal **teaching** example for reinforcement learning specifically: the state space is small enough to represent exactly (a genuine, exact lookup table, not an approximation), so every core RL concept (state, action, reward, policy, exploration) is fully visible without the added complexity a larger game would require (function approximation, neural networks) to even get off the ground.

---

# 35. Follow-up Question 11 — "Design the Actual Training Loop — Self-Play, Reward, Exploration"

> **Interviewer:** *"Walk me through one full training loop, end to end — how does the agent actually improve?"*

The agent plays a large number of complete games **against itself** (or against a mix of itself and other strategies), and after every move, updates its estimate of that move's value based on the eventual outcome — using the standard **Q-learning** update rule, which nudges the estimated value of the move just taken toward the actual reward received plus a discounted estimate of the best move available from the resulting position. Critically, the agent must **not** always pick its currently-best-known move during training (that would let it converge prematurely on a mediocre policy it never explores past) — an **epsilon-greedy** exploration policy has it pick a random move some fraction of the time, gradually reducing that fraction as training progresses.

---

# 36. Implementing Q-Learning via Self-Play

```java
public class QLearningAgent implements AiStrategy {
    private final Map<String, double[]> qTable = new HashMap<>(); // boardKey -> value per cell (NaN if illegal)
    private double epsilon = 0.1; // exploration rate, tuned down over training (§37 shows the training loop itself)

    @Override
    public int selectMove(Board board, Symbol aiSymbol) {
        if (ThreadLocalRandom.current().nextDouble() < epsilon) {
            List<Integer> emptyCells = board.emptyCellIndices();
            return emptyCells.get(ThreadLocalRandom.current().nextInt(emptyCells.size())); // EXPLORE
        }
        return bestKnownMove(board); // EXPLOIT the current policy
    }

    private int bestKnownMove(Board board) {
        double[] values = qTable.computeIfAbsent(board.canonicalKey(), k -> new double[9]);
        int best = -1;
        double bestValue = Double.NEGATIVE_INFINITY;
        for (int cell : board.emptyCellIndices()) {
            if (values[cell] > bestValue) { bestValue = values[cell]; best = cell; }
        }
        return best;
    }
}

public class QLearningTrainer {
    private static final double LEARNING_RATE = 0.3;
    private static final double DISCOUNT = 0.9;

    public void trainSelfPlay(QLearningAgent agent, int episodes) {
        for (int episode = 0; episode < episodes; episode++) {
            List<StateActionPair> history = new ArrayList<>();
            Board board = new Board();
            GameState state = new InProgressState();
            Symbol currentTurn = Symbol.X;

            while (state instanceof InProgressState) {
                int cell = agent.selectMove(board, currentTurn);
                history.add(new StateActionPair(board.canonicalKey(), cell));
                state = state.applyMove(board, new Move(currentTurn, cell, history.size()));
                currentTurn = (currentTurn == Symbol.X) ? Symbol.O : Symbol.X;
            }

            double finalReward = rewardFor(state); // +1 win, -1 loss, 0 draw, FROM THE TRAINING AGENT's perspective
            backpropagateRewards(agent, history, finalReward); // update EVERY move made this game, discounted by recency
        }
    }
}
```

`backpropagateRewards` is where the actual learning happens: the *final* game outcome is known only at the very end, but every move made throughout the game contributed to it, so the reward is propagated **backward** through the game's move history, with the `DISCOUNT` factor making moves closer to the actual win/loss count more strongly than early, more speculative moves — precisely how Q-learning turns a single win/loss signal into a gradually-refined value estimate for every position visited along the way.

---

# 37. Follow-up Question 12 — "Minimax Is Already Perfect. Why Bother Training an RL Agent at All?"

> **Interviewer:** *"You already have a provably unbeatable AI from minimax. What's the actual justification for building a trained agent as well?"*

Three genuine reasons, none of which are about raw strength (minimax already wins that comparison unconditionally): (1) **pedagogical value** — an RL agent visibly *improving* over training episodes is a much richer, more demonstrable teaching tool than a solver that's simply always already correct; (2) **tunable imperfection** — a partially-trained agent naturally plays at an *intermediate* strength between "Easy" random play and "Hard" unbeatable minimax, giving a genuinely distinct difficulty tier that isn't just minimax with artificial randomness bolted on; (3) **generality of the technique** — the exact same self-play/reward/exploration framework scales (with function approximation replacing the exact lookup table) to games far too large for exhaustive minimax, making Tic-Tac-Toe a low-stakes place to learn the technique correctly before applying it somewhere minimax genuinely isn't an option.

---

# 38. Why Train an RL Agent When a Perfect Solver Already Exists

```text
                    Strength            Determinism           Teaching value
Minimax           Provably unbeatable   Always identical,     Low (no learning to observe --
                                         no learning involved   it's simply always already correct)

Q-Learning        Depends on training    Improves visibly       High (win-rate against a fixed
(partially        episodes -- tunable    over episodes,          opponent climbs over training,
 trained)         intermediate strength  genuinely "learns"      directly demonstrable)
```

This table is the honest answer to "why not just always use minimax": minimax and a trained agent are solving two **different problems** — "what is the objectively correct move" versus "how does an agent get better through experience" — and a system offering AI difficulty tiers benefits from having both, for genuinely different reasons.

---

# 39. Class Diagram: AI Strategies

```text
+------------------------+
|       AiStrategy          |
|    <<interface>>          |
|  selectMove(board, symbol)|
+-----------+--------------+
      ^      ^      ^      ^
+----------+ +-----------+ +-----------+ +------------------------+
| Random   | | Heuristic | | Minimax   | | QLearningAgent          |
| Strategy | | Strategy  | | Strategy  | | (trained via            |
|          | |           | |           | |  QLearningTrainer, §36) |
+----------+ +-----------+ +-----------+ +------------------------+

                    +------------------------+
                    |      AiPlayer            |
                    |  decideMove(board, ...)  |
                    |  strategy: AiStrategy    |
                    +------------------------+
```

All four difficulty tiers slot into the identical `AiPlayer`/`AiStrategy` structure introduced in §25 — a new difficulty (or an entirely new algorithm, e.g., Monte Carlo tree search) is added by implementing one interface, never by modifying `AiPlayer`, `GameEngine`, or either transport session.

---

# 40. Follow-up Question 13 — "How Do You Track Scores Across Many Games and Many Players?"

> **Interviewer:** *"A single game's result is easy. How do you turn that into persistent, per-player win/loss/draw records and a leaderboard?"*

Every completed game (regardless of whether it was local, networked, against another human, or against an AI) should emit a single, immutable **game result event** — winner (or draw), both players' identities, timestamp — that a `ScoreRepository` durably records and incrementally folds into each player's running totals, exactly the same "compute once, persist immediately, never re-derive" discipline this series has applied to every other guide's event-sourced state.

---

# 41. Score Management and Leaderboards

```text
GameEngine reaches a terminal state (X_WINS/O_WINS/DRAW)
        |
        v
ScoreRepository.recordResult(playerX, playerO, result)
        |
        +--> playerX's record: wins/losses/draws counters incremented appropriately
        +--> playerO's record: wins/losses/draws counters incremented appropriately
        |
        v
Leaderboard: a read-optimized VIEW over all players' records, sorted by win count
             (or a more nuanced rating, e.g. win percentage weighted by games played)
```

Separating the **write path** (recording one game's result, which happens once per completed game) from the **read path** (rendering a leaderboard, which happens far more often, every time any player views it) mirrors the same read/write-path separation the Parking Lot guide's §43 and the Real-Time Analytics Platform guide's hot/cold store split both already established: frequent reads should never be served by re-scanning or re-aggregating from the write-side records on every request.

---

# 42. Implementing the Score Repository

```java
public class ScoreRepository {
    private final Map<String, PlayerRecord> recordsByPlayerId = new ConcurrentHashMap<>();

    public void recordResult(Player playerX, Player playerO, GameResult result) {
        PlayerRecord xRecord = recordsByPlayerId.computeIfAbsent(playerX.playerId(), id -> new PlayerRecord(id));
        PlayerRecord oRecord = recordsByPlayerId.computeIfAbsent(playerO.playerId(), id -> new PlayerRecord(id));

        switch (result) {
            case X_WINS -> { xRecord.incrementWins(); oRecord.incrementLosses(); }
            case O_WINS -> { oRecord.incrementWins(); xRecord.incrementLosses(); }
            case DRAW -> { xRecord.incrementDraws(); oRecord.incrementDraws(); }
            case IN_PROGRESS -> throw new IllegalArgumentException("Cannot record an unfinished game");
        }
    }

    public List<PlayerRecord> leaderboard(int topN) {
        return recordsByPlayerId.values().stream()
            .sorted(Comparator.comparingInt(PlayerRecord::wins).reversed())
            .limit(topN)
            .toList();
    }
}
```

Rejecting `IN_PROGRESS` results explicitly (rather than silently no-oping) is a small but deliberate defensive detail — it converts a caller's genuine bug (recording a score before the game actually ended) into an immediate, loud failure rather than a silently-corrupted leaderboard that would be far harder to trace back to its cause later.

---

# 43. Follow-up Question 14 — "How Do You Support a Contest With a Prize, Across a Match Series?"

> **Interviewer:** *"Instead of a single game, design a 'best of 5' match series between two players, where the overall series winner receives a prize."*

A `MatchSeries` aggregates a configured number of individual games between the same two players, tracking each game's result and determining an overall winner once one player reaches the majority threshold (e.g., 3 wins out of a best-of-5) — reusing `GameEngine` unchanged for every individual game within the series, since a series is simply a sequence of otherwise-ordinary games with an additional aggregation layer on top, not a different kind of game.

---

# 44. Designing Match Series and Contests

```java
public class MatchSeries {
    private final Player playerA;
    private final Player playerB;
    private final int gamesToWin; // e.g., 3 for a best-of-5 series
    private int playerAWins = 0;
    private int playerBWins = 0;
    private final List<GameResult> completedGames = new ArrayList<>();

    public void recordGameResult(GameResult result) {
        completedGames.add(result);
        if (result == GameResult.X_WINS) playerAWins++;   // assuming playerA is always X for this series
        else if (result == GameResult.O_WINS) playerBWins++;
    }

    public Optional<Player> seriesWinner() {
        if (playerAWins >= gamesToWin) return Optional.of(playerA);
        if (playerBWins >= gamesToWin) return Optional.of(playerB);
        return Optional.empty(); // series still in progress
    }
}

public class Contest {
    private final BigDecimal prizePool;
    private final MatchSeries series;

    public void distributePrizeIfComplete(PrizeLedger ledger) {
        series.seriesWinner().ifPresent(winner -> ledger.credit(winner.playerId(), prizePool));
    }
}
```

`Contest` deliberately holds a reference to `MatchSeries` rather than reimplementing win-counting itself — the prize-distribution concern is layered cleanly on top of the already-correct series logic, exactly the same compositional relationship this series has used repeatedly (a Decorator or a thin wrapper adding one new concern over an already-correct lower layer) rather than duplicating logic across the two.

---

# 45. Implementing Contest and Prize Distribution

The `PrizeLedger.credit(...)` call above is the one operation in this entire guide that must be treated with the same rigor as an actual financial transaction — it should be idempotent (crediting the same completed series twice must not double-pay), auditable (every credit permanently logged with the series/contest ID that justified it), and executed exactly once per concluded contest, typically enforced via the same **Idempotency-Key**-style mechanism this series' REST API guide detailed in its own §48-50: the contest's own ID serves as the idempotency key, so a retried or duplicated "distribute prize" call recognizes it has already paid out and does not pay again.

---

# 46. Follow-up Question 15 — "How Do You Prevent Cheating in a Prize-Money Contest?"

> **Interviewer:** *"Real money is now on the line. What stops a player from tampering with their client to report a fake win?"*

Precisely the principle already established back in §20-23: **the server, never either client, is the sole source of truth for game state and results** — a contest-mode game must run its `GameEngine` server-side, with each client only ever sending proposed moves and receiving authoritative board state back, exactly like `NetworkedGameSession`. A client claiming "I won" with no corresponding server-validated sequence of legal moves reaching a genuine terminal state is simply never trusted, structurally, not merely detected after the fact by a separate anti-cheat check.

---

# 47. Contest Integrity: Server-Authoritative Validation and Anti-Cheat

```text
UNTRUSTED (never do this for a real-money contest):
  Client plays a full game locally --> client sends "I WON" to the server --> server pays out
  -- a modified client can simply SEND A FALSE CLAIM, no actual game required

TRUSTED (this guide's design):
  Every move is sent to and VALIDATED by the server's own GameEngine (§21-23)
  The server's OWN GameState reaching X_WINS/O_WINS is the ONLY thing that can trigger
  MatchSeries.recordGameResult() and, eventually, Contest.distributePrizeIfComplete()
  -- a client can send whatever it wants, but only a genuinely legal, server-validated
     sequence of moves can ever produce a payable result
```

This is the same architectural lesson the Rate Limiter guide's distributed coordination (§32-35 there) and this guide's own §22-23 already established, applied to its highest-stakes form: **whatever a client reports about itself is data, not fact, the instant real value depends on the outcome** — the server must independently derive the result from validated inputs, never accept a client's self-reported conclusion.

---

# 48. Capacity Estimation: Concurrent Matches and Matchmaking

```text
Assume: 100,000 concurrent networked games at peak, each session holding a tiny amount of state
        (a Board's two integer bitmasks, a GameState reference, two connection handles)

Memory per session ≈ 200 bytes (board + state + metadata) --> 100,000 * 200 bytes = 20 MB total
  -- trivially small; a single well-provisioned server process can hold every active session's
     state in memory simultaneously, with capacity to spare for orders of magnitude more sessions

Move throughput: even at a (generously high) 10 moves/sec per active session, 100,000 sessions
  generate only 1,000,000 moves/sec platform-wide -- each move's processing cost (bitmask update,
  8-line win check) is on the order of nanoseconds, so CPU is never the bottleneck; the actual
  limiting factor at real scale is WebSocket connection count and network I/O, not game logic.
```

Tic-Tac-Toe's absolute computational cheapness means this design's scaling story is almost entirely about connection/session management infrastructure (matchmaking queues, WebSocket connection pooling) rather than anything intrinsic to the game logic itself — a deliberate contrast with this series' other guides (the Analytics Platform, the Rate Limiter), where the domain logic itself was the primary scaling concern.

---

# 49. Full Worked Example: One Networked Match, Traced End to End

```text
1. Player Alice (human) and Player Bob (human) are matched into a NetworkedGameSession,
   server-side GameEngine created with a fresh InProgressState and empty Board (§21)
2. Alice's client sends Move(X, cellIndex=4, moveNumber=1) over WebSocket
3. Server: NetworkedGameSession.onMoveReceived validates Alice is indeed playing X (§21)
4. Server: GameEngine.makeMove delegates to InProgressState.applyMove -- cell 4 was empty,
   legal -- board.place(4, X) updates the bitmask, no win yet, board not full -- state stays
   IN_PROGRESS (§17)
5. Server broadcasts the updated board snapshot to BOTH Alice's and Bob's clients (§21)
6. ... several more moves alternate, each following the identical steps 2-5 ...
7. Bob's move at cellIndex=6 completes a diagonal for O -- InProgressState.applyMove detects
   board.hasWon(O) -- returns a NEW OWinsState (§16-17)
8. Server broadcasts the final board snapshot, now tagged with result=O_WINS
9. ScoreRepository.recordResult(alice, bob, O_WINS) -- Bob's win count increments, Alice's
   loss count increments (§42)
10. If this game was part of an active MatchSeries: MatchSeries.recordGameResult(O_WINS) --
    Bob's series win count increments; if this reaches the gamesToWin threshold, Contest.
    distributePrizeIfComplete() credits Bob's PrizeLedger exactly once (§44-45)
```

Every mechanism introduced by a follow-up question in this guide appears somewhere in this one match's trace — the state machine, server-authoritative validation, score recording, and prize distribution are not independent, optional features, they are the actual steps one real networked, prize-bearing match passes through in this design.

---

# 50. Final Architecture Diagram

```text
Alice's client <--WebSocket--> [ NetworkedGameSession ] <--WebSocket--> Bob's client
                                          |
                                +---------v---------+
                                |     GameEngine       |
                                | GameState + Board    |
                                +---------+---------+
                                          |
                          +---------------+---------------+
                          v                                 v
              +------------------------+       +------------------------+
              |    ScoreRepository       |       |   MatchSeries / Contest |
              |    Leaderboard           |       |   PrizeLedger            |
              +------------------------+       +------------------------+

(Local/offline mode, structurally identical GameEngine, no network layer:)
Human/AI move --> [ LocalGameSession ] --> [ GameEngine ] --> ScoreRepository (local persistence)
                          ^
                          |
                 [ AiPlayer -> AiStrategy: Random | Heuristic | Minimax | QLearningAgent ]
```

---

# 51. Design Patterns Used Throughout This Guide

- **State** — `GameState` (§16-17) encodes exactly which move transitions are legal from the current phase, structurally preventing illegal moves rather than relying on scattered runtime checks.
- **Strategy** — `AiStrategy` (§25-39: random, heuristic, minimax, Q-learning) is a single swappable interface behind which every AI difficulty tier lives, interchangeable via configuration alone.
- **Facade** — `GameEngine` hides board mutation, win detection, and state transition behind a single `makeMove` entry point both transport layers and both AI adapters depend on.
- **Adapter** — the thin `AiPlayer` wrapper adapts an `AiStrategy`'s move selection into the exact same `Move` shape a human client submits, letting `GameEngine` treat both uniformly (§24-25).
- **Repository** — `ScoreRepository` and `PrizeLedger` each encapsulate their own persistence and aggregation concerns behind a narrow, purpose-specific interface.

---

# 52. SOLID Principles Applied

- **Single Responsibility** — `Board` only tracks cell occupancy and win detection; `GameState` only encodes legal transitions; `ScoreRepository` only aggregates results — none of the three knows how to do the others' job, even though `GameEngine` composes the first two on every move.
- **Open/Closed** — adding a new AI difficulty (or a new contest format) means implementing one interface, never modifying `GameEngine`, either transport session, or `MatchSeries`.
- **Liskov Substitution** — every `AiStrategy` implementation must honestly return a legal, currently-empty cell index from `selectMove`, so `AiPlayer` can compose any of them interchangeably without special-casing a particular strategy.
- **Interface Segregation** — `AiStrategy` exposes exactly one method, so a trivial `RandomStrategy` isn't forced to depend on concepts (Q-tables, alpha-beta bounds) only more elaborate strategies actually need.
- **Dependency Inversion** — both `LocalGameSession` and `NetworkedGameSession` depend only on the `GameEngine` class's public contract, never reimplementing rules themselves, so the exact same rules engine serves both transports without duplication.

---

# 53. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Re-scanning all 8 winning lines cell-by-cell every move  Precomputed bitmask win detection (§12-14)
A boolean isGameOver flag with scattered if-checks        Explicit State pattern with legal transitions (§16-17)
Duplicating game rules in both local and networked code   One GameEngine, two thin transport layers (§19-21)
Trusting a client's self-reported win in a networked game Server-authoritative GameEngine (§22-23)
Special-casing AI moves differently from human moves      AiPlayer produces the same Move shape (§24-25)
Confusing "unbeatable" (minimax) with "trained" (RL)       Understand solving vs. learning are different (§33-34)
Letting an RL agent always exploit during training         Epsilon-greedy exploration (§35-36)
Paying out a contest prize based on a client's claim        Server derives the result independently (§46-47)
```

---

# 54. Testing Strategy

- **Win detection tests** — verify every one of the 8 winning-line bitmasks correctly triggers `hasWon`, and that a non-winning configuration correctly does not.
- **State machine tests** — assert every illegal transition (move after game end, move on an occupied cell, move out of turn) throws, and every legal transition produces the correct resulting state.
- **Minimax correctness tests** — play the minimax strategy against itself and assert every game ends in a draw (the mathematically guaranteed outcome for two optimal Tic-Tac-Toe players); play it against every other strategy and assert it never loses.
- **Q-learning convergence tests** — train an agent for a large number of self-play episodes and assert its win rate against a fixed heuristic opponent measurably improves over the course of training, confirming the learning loop is actually learning, not merely running.
- **Contest integrity tests** — simulate a client submitting an illegal move sequence or a forged "I won" claim directly to the server, and assert the server's own `GameEngine` state (not the client's claim) is the only thing that can trigger prize distribution.

---

# 55. Suggested Future Enhancements

- **Larger board variants** (4x4, 5x5 with a run-of-4 to win) — the bitmask technique from §12-14 extends naturally to a wider integer type, though minimax's exhaustive search (§30-32) would need a depth limit and a heuristic evaluation function once the game tree grows too large to fully explore.
- **ELO-style rating** instead of raw win counts, so the leaderboard (§41-42) reflects opponent strength, not just win volume.
- **Bracket-style tournaments** among many players, generalizing the two-player `MatchSeries` (§44) into a multi-round elimination structure.
- **Spectator mode** for networked matches, broadcasting board state to read-only observers in addition to the two active players.
- **Neural-network function approximation** replacing the Q-learning agent's exact table (§36) — directly relevant once this guide's techniques are applied to a board too large to enumerate exactly, per §37's third justification for training at all.

---

# 56. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Design a board representation and win-check that avoids scanning all 8 lines cell-by-cell on every move. (§12-14)
2. Model the game's lifecycle so an illegal move (wrong turn, occupied cell, move after game end) is structurally rejected. (§15-17)
3. Design the game engine so the identical rules serve both local pass-and-play and real-time networked multiplayer. (§19-21)
4. Design an AI opponent that is provably unbeatable, and explain precisely why it can be proven so. (§30-32)
5. The prompt asks you to "train" an AI. What does that actually mean for a fully-solved game like this, and how would you implement it? (§33-36)
6. If minimax is already unbeatable, what's the actual justification for also building a trained learning agent? (§37-38)
7. Design a prize-bearing contest across a match series, and explain specifically how you prevent a malicious client from claiming a false win. (§43-47)
8. Estimate the resource cost of supporting 100,000 concurrent networked games, and identify the actual bottleneck at that scale. (§48)

---

# 57. Final Takeaway

Every hard decision in this guide traces back to one recurring idea: **keep the small number of truly hard problems isolated behind narrow interfaces, so the trivial parts of the system never have to know about them** — `GameEngine` doesn't know or care whether a move came from a human, a minimax solver, or a Q-learning agent (§24-25); neither transport layer reimplements game rules (§19-21); and no client, however sophisticated its tampering, can influence a result the server itself didn't independently derive (§46-47). Tic-Tac-Toe's trivial rules make every one of these boundaries fully visible — which is exactly why building the *system* around it, not merely the game, is worth taking seriously.

---

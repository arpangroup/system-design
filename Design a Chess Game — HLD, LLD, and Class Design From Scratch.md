# 1. What We Are Building

A two-player chess engine and game service: a board that enforces every rule of chess (piece movement, check, checkmate, stalemate, castling, en passant, pawn promotion, and the quieter draw conditions), a move history that supports undo/redo, and an optional AI opponent that picks a move by searching ahead rather than guessing. The same core engine should work whether the two players are sitting at one screen, playing over a network, or playing against the computer.

Chess looks like a "just implement the rules" problem, and that's exactly the trap: the rules are large, mutually interacting, and full of classic correctness bugs (a piece that can legally move to a square except that doing so would leave its own king in check; a pawn that captures a square it never landed on, via en passant; a king that can castle except that one of the squares it passes through is currently attacked). We will build this from first principles: model the board and pieces so move generation is pluggable per piece type, model check/checkmate/stalemate as derived facts computed from move generation rather than special-cased separately, and design the AI opponent around a search algorithm whose correctness doesn't depend on how deep it happens to look.

---

# 2. Learning Objectives

By the end of this guide, you should be able to:

- Represent a chess board and its pieces so that adding or changing a piece's movement rule never requires touching unrelated code.
- Generate legal moves in two clean phases — pseudo-legal moves per piece, then a self-check filter — and explain why collapsing these into one phase is where the classic "moving into check" bug comes from.
- Detect check, checkmate, and stalemate as three closely related derived facts, not three independently-implemented special cases.
- Implement castling and en passant as move *types* validated by their own preconditions, without special-casing every other rule to know about them.
- Implement undo/redo cleanly using the Command pattern, storing exactly what's needed to reverse a move — including the tricky cases (a captured piece, a pawn's en passant rights, a king or rook's castling rights).
- Detect the three draw conditions (threefold repetition, the fifty-move rule, insufficient material) without re-deriving game history from scratch on every check.
- Explain and implement minimax with alpha-beta pruning as the AI's move-selection algorithm, and explain why alpha-beta produces the *same* result as plain minimax while doing dramatically less work.
- Identify the same "two clients racing to submit a move" correctness problem this series has repeatedly solved for seats, cash, and contest slots, now applied to a shared game state.

---

# 3. Why This Matters (The Interview, Framed)

"Design a chess game" sounds like a pure implementation exercise — surely you just encode the rules? — but the rules of chess are exactly rich enough to test whether a candidate's design instincts generalize, or whether they only work for problems they've memorized.

> **Interviewer:** *"How do you know if a move is legal?"*

A weak answer describes checking whether the destination square is reachable by the piece's movement pattern and stops there. A strong answer immediately volunteers the second, easy-to-forget half: a move can be reachable by the piece's pattern and *still* be illegal, if making it would leave the mover's own king in check. Recognizing that legality is a two-phase computation — not a single lookup — is the entire signal this question is designed to surface, and it's the same signal every subsequent question in this guide keeps testing from a different angle: castling legality, en passant legality, checkmate detection, even AI move selection all hinge on getting this two-phase structure right once and reusing it everywhere, rather than re-deriving a slightly different version of it for each new rule.

---

# 4. Recommended Technology Stack

| Layer | Choice | Why |
|---|---|---|
| Core engine | Java / Kotlin / TypeScript (language-agnostic design) | Move generation and check detection are pure, deterministic logic — the interesting design decisions are structural, not tied to a specific runtime. |
| Board representation | 8×8 array of nullable `Piece` references, or a 0x88/bitboard for a performance-focused engine | An object array keeps the design approachable and is what this guide builds; bitboards are the standard upgrade path once raw move-generation speed matters (deep AI search). |
| Client rendering | Any component framework, or plain Canvas/SVG for a simple board | The board's visual state is a pure function of engine state — nothing about rendering should leak back into rule enforcement. |
| Multiplayer transport | WebSocket | A game is a long-lived, low-latency, bidirectional exchange of small messages — exactly what a persistent socket is for, versus repeated request/response polling. |
| Game state persistence | Relational DB (moves table) or an event-sourced log of moves | Storing the *move list*, not just the current board snapshot, is what makes undo, replay, and PGN export possible without extra bookkeeping. |
| AI search | In-process minimax + alpha-beta, optionally offloaded to a worker thread/process | Search is CPU-bound and blocking; keeping it off the main request-handling thread avoids stalling other players' moves. |
| Opening/endgame data (optional) | A small opening-book lookup table and tablebase for late-game positions | Bounds how deep the AI needs to search from well-known starting positions, without changing the search algorithm itself. |

---

# 5. Project Structure

```text
chess-engine/
├── board/
│   ├── Board.java
│   ├── Square.java
│   └── Piece.java                            -- §10
├── movegen/
│   ├── MoveGenerator.java                     -- §17-18
│   ├── PieceMoveStrategy.java                 -- §17
│   ├── PawnMoveStrategy.java                  -- §17-18
│   ├── KnightMoveStrategy.java                -- §17-18
│   ├── SlidingMoveStrategy.java                -- §17-18 (bishop/rook/queen share this)
│   └── KingMoveStrategy.java                  -- §17-18
├── rules/
│   ├── CheckDetector.java                     -- §20-21
│   ├── LegalMoveFilter.java                    -- §25
│   ├── SpecialMoveRules.java                   -- §22-23
│   └── DrawConditionDetector.java              -- §30-31
├── game/
│   ├── GameState.java                          -- §13-14
│   ├── MoveHistory.java                        -- §27-29
│   └── MoveCommand.java                        -- §28-29
├── ai/
│   ├── MinimaxSearch.java                      -- §33-35
│   ├── AlphaBetaSearch.java                    -- §34-35
│   └── BoardEvaluator.java                     -- §36-37
├── multiplayer/
│   └── ConcurrentMoveSubmission.java           -- §38-39
└── api/
    └── GameController.java
```

---

# 6. Follow-up Question 1: What Are the Core Nouns Here, Before We Draw Any Boxes?

> **Interviewer:** *"Before you design anything, what are the core entities?"*

The distinction worth making explicit up front: a `Move` is not the same thing as a `MoveCommand` — a `Move` is a plain description of what happens (from-square, to-square, piece, maybe a captured piece), while a `MoveCommand` (§28) is the *executable, reversible* action built from a `Move`. Conflating the two is a common reason undo ends up bolted on later instead of designed in from the start.

---

# 7. Functional Requirements

- The system must generate the full set of legal moves for any piece on any square, correctly excluding moves that would leave the mover's own king in check.
- The system must detect check, checkmate, and stalemate correctly and immediately after every move.
- The system must correctly implement all special moves: kingside/queenside castling (with all of its preconditions), en passant capture, and pawn promotion (to any of the four promotable piece types).
- The system must detect all three draw conditions: threefold repetition, the fifty-move rule, and insufficient mating material.
- The system must support undo and redo of any number of moves, correctly restoring captured pieces, castling rights, and en passant eligibility.
- The system must support an AI opponent that selects a move by searching ahead a configurable number of plies, at a difficulty level a human can tune.
- The system must support two remote players submitting moves over a network without either player's move being silently lost, duplicated, or applied out of turn.
- The system must be able to export and import a game as a standard move list (PGN-style) for replay.

---

# 8. Non-Functional Requirements

- **Correctness above all**: an illegal move must never be accepted, and a legal move must never be rejected — chess has zero tolerance for "mostly right" rule enforcement, because a single wrong ruling invalidates the entire game.
- **Determinism**: given an identical sequence of moves, the engine must always reach an identical resulting state — this is what makes replay, undo, and multiplayer synchronization possible at all.
- **Reasonable AI response latency**: the AI must return a move within an interactively acceptable time budget (sub-second to a few seconds depending on configured strength), which shapes the search algorithm choice as much as the evaluation function does.
- **No state corruption under concurrent submission**: two near-simultaneous move submissions for the same game must never both be applied — exactly one succeeds, and turn order is preserved.

---

# 9. Follow-up Question 2: Why Not Just Represent the Board as an 8×8 Array of Characters Like `'P'`, `'k'`, `'.'`?

> **Interviewer:** *"What's wrong with a simple character grid for the board?"*

A character grid works fine for *rendering* but starts fighting you the moment you need to attach state to a piece rather than just its type — has this particular rook moved yet (castling rights), is this particular pawn eligible to be captured en passant *this* turn only, which specific piece instance was captured by which move (for undo). A character is a value with no identity; a chess piece over the course of a game needs identity and mutable-but-trackable state, which is exactly what an object (or an equivalent struct with an explicit identity) gives you for free.

---

# 10. Identifying the Core Domain Entities

```text
Board          -- an 8x8 grid of Squares, each holding at most one Piece
Square         -- one coordinate on the board (file, rank), e.g. e4
Piece          -- a single chess piece instance: type, color, hasMoved flag, current Square
Move           -- a plain description: fromSquare, toSquare, piece, capturedPiece (nullable), moveType
MoveCommand    -- an executable, reversible wrapper around a Move (§28)
GameState      -- whose turn it is, castling rights, en passant target square, halfmove clock, full move number
MoveHistory    -- the ordered list of MoveCommands played so far, enabling undo/redo and repetition detection
Player         -- a human or an AI, identified by color (White/Black)
```

The two entities most often merged by mistake are `Move` and `MoveCommand` (flagged in §6) and `Board` and `GameState` — the board is *which piece is on which square*; the game state is everything else needed to correctly judge legality that isn't spatial (can this side still castle kingside, is there an en passant capture available this exact turn). Keeping them separate means a function that only needs spatial information doesn't have to be handed the entire game state.

---

# 11. High-Level Architecture Overview

```text
                       ┌───────────────┐
                       │   Client (UI)  │
                       └───────┬────────┘
                               │ submit Move
                       ┌───────▼────────┐
                       │ Game Controller │──── concurrency guard (§38-39)
                       └───────┬────────┘
                               │
                ┌──────────────▼───────────────┐
                │         GameState              │
                │  (turn, castling rights,        │
                │   en passant target, clocks)     │
                └──────────────┬───────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────▼────────┐   ┌─────────▼─────────┐   ┌─────────▼─────────┐
│  MoveGenerator   │   │  CheckDetector       │   │ DrawConditionDet.   │
│  (§17-18)        │   │  (§20-21)            │   │ (§30-31)            │
└───────┬────────┘   └─────────┬─────────┘   └─────────┬─────────┘
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                       ┌───────▼────────┐
                       │ LegalMoveFilter  │  (§25 -- self-check exclusion)
                       └───────┬────────┘
                               │
                       ┌───────▼────────┐
                       │   MoveHistory    │──── undo/redo (§27-29)
                       └───────┬────────┘
                               │
                       ┌───────▼────────┐
                       │      Board       │
                       └────────────────┘

  Optional AI player: MinimaxSearch/AlphaBetaSearch (§33-35) calls the
  same MoveGenerator + LegalMoveFilter a human move would go through --
  the AI never gets a privileged shortcut around rule enforcement.
```

The architectural rule worth stating explicitly: the AI opponent is just another caller of the exact same legal-move-generation pipeline a human move goes through. It doesn't get its own, parallel implementation of "what's legal here" — it only adds a search-and-evaluate layer on top, which is what guarantees the AI can never accidentally play an illegal move.

---

# 12. Follow-up Question 3: Why Model the Game as an Explicit State Machine Instead of a Handful of Booleans (`isCheck`, `isOver`, etc.)?

> **Interviewer:** *"Why not just track a few boolean flags for game status?"*

Booleans compose badly — `isCheck && isOver` and `isCheck && !isOver` are both representable, but only one of the four combinations of two independent-looking booleans is ever actually reachable at a time in real chess (checkmate implies both check and over; stalemate implies over but not check). An explicit state machine makes the *reachable* combinations the only representable ones, which is exactly the same reasoning this series has applied to every other lifecycle so far — a contest, a parking spot, an elevator car, a vending machine's dispense cycle.

---

# 13. The Game Lifecycle as a State Machine

```text
   ┌─────────┐  move played,   ┌──────────┐  move played,     ┌────────────┐
   │ IN_PROGRESS │────────────▶│ IN_CHECK   │──────────────────▶│ IN_PROGRESS │
   └─────┬───┘  puts opponent  └────┬─────┘  check resolved     └────────────┘
         │      in check            │
         │                          │ no legal move escapes check
         │                          ▼
         │                   ┌────────────┐
         │                   │ CHECKMATE   │  (terminal)
         │                   └────────────┘
         │
         │ side to move has no legal move, and is NOT in check
         ▼
   ┌────────────┐
   │  STALEMATE  │  (terminal)
   └────────────┘

   Also reachable from IN_PROGRESS directly (terminal, no check involved):
   THREEFOLD_REPETITION | FIFTY_MOVE_RULE | INSUFFICIENT_MATERIAL | RESIGNATION | DRAW_AGREED
```

Every one of these states — including check itself — is *computed*, never separately tracked and separately updated. After each move, the engine asks the `CheckDetector` and `DrawConditionDetector` fresh questions against the current board and history; it never maintains an `isCheck` flag that some code path could forget to update. This is the same discipline as the ATM guide's `Account.availableCents()` being a derived computation rather than a separately-tracked field.

---

# 14. Implementing the State Pattern for Game State

```java
public enum GameStatus {
    IN_PROGRESS, CHECKMATE, STALEMATE,
    THREEFOLD_REPETITION, FIFTY_MOVE_RULE, INSUFFICIENT_MATERIAL,
    RESIGNATION, DRAW_AGREED
}

public class GameState {
    private Board board;
    private Color sideToMove;
    private CastlingRights castlingRights;
    private Square enPassantTarget;   // null unless the last move was a two-square pawn advance
    private int halfmoveClock;        // moves since the last capture or pawn move -- §30
    private int fullmoveNumber;

    public GameStatus computeStatus(CheckDetector checkDetector, DrawConditionDetector drawDetector) {
        boolean inCheck = checkDetector.isInCheck(board, sideToMove);
        boolean hasLegalMove = !LegalMoveFilter.legalMovesFor(board, this, sideToMove).isEmpty();
        if (!hasLegalMove) {
            return inCheck ? GameStatus.CHECKMATE : GameStatus.STALEMATE;
        }
        return drawDetector.checkDrawConditions(this).orElse(GameStatus.IN_PROGRESS);
    }
}
```

`computeStatus` is deliberately the *only* place that decides checkmate vs. stalemate vs. in-progress — every caller (the API layer, the AI, the test suite) asks this one method rather than re-deriving the logic, which is exactly what prevents the two states from ever drifting out of sync with each other.

---

# 15. Class Diagram: The Game Core

```text
┌────────────────┐        ┌───────────────────┐
│    GameState      │◀──────│      MoveHistory      │
│ ────────────────│  1   1 │ ───────────────────│
│ board              │        │ commands: List<...>   │
│ sideToMove         │        │ recordAndExecute()    │
│ castlingRights     │        │ undo()                │
│ enPassantTarget    │        │ redo()                │
│ halfmoveClock       │        └───────────────────┘
│ computeStatus()    │
└─────────┬────────┘
          │ 1
          │
          ▼ 1
┌────────────────┐        ┌───────────────────┐
│      Board        │◀──────│       Square          │
│ ────────────────│  1  64 │ ───────────────────│
│ squares[8][8]       │        │ file, rank             │
│ pieceAt(square)     │        └───────────────────┘
└─────────┬────────┘
          │ 0..1 per Square
          ▼
┌────────────────┐
│      Piece         │
│ ────────────────│
│ type, color         │
│ hasMoved            │
│ currentSquare       │
└────────────────┘
```

---

# 16. Follow-up Question 4: How Do You Generate Moves for Six Different Piece Types Without a Giant `switch` Statement?

> **Interviewer:** *"A pawn moves differently from a knight, which moves differently from a bishop. How do you structure this?"*

Each piece type has its own, self-contained movement rule, and none of them need to know about the others — this is exactly the shape the Strategy pattern exists for. A `switch` on piece type buried inside one `generateMoves` method means every new special case (a pawn's double-first-move, a knight's jump, a sliding piece's "stop at the first blocker") all live in one function's worth of tangled conditionals; one `PieceMoveStrategy` per piece type means each rule is independently readable, independently testable, and — critically — independently *wrong or right*, without any risk of one piece's bug leaking into another's logic.

---

# 17. Move Generation as a Pluggable Strategy

```java
public interface PieceMoveStrategy {
    List<Move> pseudoLegalMoves(Board board, Square from, Piece piece);
}

public class SlidingMoveStrategy implements PieceMoveStrategy {
    private final List<int[]> directions; // e.g. rook: {0,1},{0,-1},{1,0},{-1,0}

    public List<Move> pseudoLegalMoves(Board board, Square from, Piece piece) {
        List<Move> moves = new ArrayList<>();
        for (int[] dir : directions) {
            Square current = from;
            while (true) {
                current = current.offsetBy(dir[0], dir[1]);
                if (!current.isOnBoard()) break;
                Piece occupant = board.pieceAt(current);
                if (occupant == null) {
                    moves.add(Move.quiet(from, current, piece));
                } else {
                    if (occupant.color() != piece.color()) {
                        moves.add(Move.capture(from, current, piece, occupant));
                    }
                    break; // blocked -- whether by a friendly piece or a just-captured enemy piece
                }
            }
        }
        return moves;
    }
}
```

`SlidingMoveStrategy` is reused for the bishop, rook, and queen, parameterized only by direction set — the queen's `directions` list is literally the bishop's plus the rook's. This is the same "shared shape, different parameters" idea this series used for `SportRuleSet` — one interface, several small parameterized implementations, zero duplicated control flow.

---

# 18. Implementing the Remaining Piece Move Strategies

```java
public class PawnMoveStrategy implements PieceMoveStrategy {
    public List<Move> pseudoLegalMoves(Board board, Square from, Piece piece) {
        List<Move> moves = new ArrayList<>();
        int forward = piece.color() == Color.WHITE ? 1 : -1;
        Square oneStep = from.offsetBy(0, forward);
        if (oneStep.isOnBoard() && board.pieceAt(oneStep) == null) {
            moves.add(Move.quiet(from, oneStep, piece));
            boolean onStartingRank = (piece.color() == Color.WHITE && from.rank() == 2)
                                   || (piece.color() == Color.BLACK && from.rank() == 7);
            Square twoStep = from.offsetBy(0, forward * 2);
            if (onStartingRank && board.pieceAt(twoStep) == null) {
                moves.add(Move.twoSquareAdvance(from, twoStep, piece));  // enables en passant next turn -- §22-23
            }
        }
        for (int dx : new int[]{-1, 1}) {
            Square diag = from.offsetBy(dx, forward);
            if (!diag.isOnBoard()) continue;
            Piece occupant = board.pieceAt(diag);
            if (occupant != null && occupant.color() != piece.color()) {
                moves.add(Move.capture(from, diag, piece, occupant));
            }
        }
        return moves; // en passant and promotion are layered on separately -- §22-23
    }
}

public class KnightMoveStrategy implements PieceMoveStrategy {
    private static final int[][] OFFSETS = {{1,2},{2,1},{-1,2},{-2,1},{1,-2},{2,-1},{-1,-2},{-2,-1}};
    public List<Move> pseudoLegalMoves(Board board, Square from, Piece piece) {
        List<Move> moves = new ArrayList<>();
        for (int[] o : OFFSETS) {
            Square target = from.offsetBy(o[0], o[1]);
            if (!target.isOnBoard()) continue;
            Piece occupant = board.pieceAt(target);
            if (occupant == null) moves.add(Move.quiet(from, target, piece));
            else if (occupant.color() != piece.color()) moves.add(Move.capture(from, target, piece, occupant));
        }
        return moves;
    }
}
```

Notice `pseudoLegalMoves` — the name is deliberate. These strategies answer "where can this piece's movement pattern reach," not "where can this piece *legally* move" — that second, stricter question needs the self-check filter from §25, which is why it's a separate phase rather than folded into each strategy.

---

# 19. Follow-up Question 5: How Do You Correctly Detect Check, Checkmate, and Stalemate — Without Three Separate Implementations?

> **Interviewer:** *"Walk me through how you'd detect checkmate."*

All three conditions reduce to two underlying computations already built for other reasons: "is the king currently attacked" (check) and "does the side to move have any legal move at all" (used to distinguish checkmate from stalemate, and — with the check flag — to decide which one applies). Neither needs its own bespoke algorithm; both are compositions of move generation and attack detection that already exist.

---

# 20. Check Detection and King Safety

"Is square X attacked by color C" is answered by asking a much simpler, related question: could any piece of color C move a queen there right now, or a knight, or a pawn on a diagonal — i.e., generate every pseudo-legal move for the attacking side and check whether any of them lands on the target square. This reuses `MoveGenerator` (§17-18) directly, rather than writing a second, parallel "attack map" algorithm that has to be kept consistent with the first.

```java
public class CheckDetector {
    public boolean isSquareAttacked(Board board, Square square, Color byColor) {
        for (Square from : board.occupiedSquaresFor(byColor)) {
            Piece piece = board.pieceAt(from);
            List<Move> pseudoLegal = moveGenerator.pseudoLegalMovesFor(board, from, piece);
            if (pseudoLegal.stream().anyMatch(m -> m.to().equals(square))) return true;
        }
        return false;
    }

    public boolean isInCheck(Board board, Color color) {
        Square kingSquare = board.findKing(color);
        return isSquareAttacked(board, kingSquare, color.opposite());
    }
}
```

---

# 21. Implementing Checkmate and Stalemate Detection

```java
public GameStatus resolveTerminalStatus(Board board, GameState state, Color sideToMove) {
    boolean inCheck = checkDetector.isInCheck(board, sideToMove);
    List<Move> legalMoves = legalMoveFilter.legalMovesFor(board, state, sideToMove);  // §25
    if (legalMoves.isEmpty()) {
        return inCheck ? GameStatus.CHECKMATE : GameStatus.STALEMATE;
    }
    return GameStatus.IN_PROGRESS;
}
```

This is the entire implementation — because `legalMovesFor` (§25) already excludes any move that leaves the mover's own king in check, "no legal moves and currently in check" and "no legal moves and not in check" are the only two ways to have zero legal moves, and they are exactly checkmate and stalemate by chess's own definitions. There is no separate checkmate-detection algorithm to get subtly wrong.

---

# 22. Follow-up Question 6: How Do You Implement Castling and En Passant Without Special-Casing Every Other Rule to Know About Them?

> **Interviewer:** *"Castling has like four preconditions. How do you keep that from leaking into your general move-validation code?"*

Both castling and en passant are modeled as their own distinct `MoveType`, each validated by its own, self-contained precondition checker that runs *only* when that move type is being considered — the general move-generation and legality pipeline stays entirely unaware of their existence beyond "here is one more candidate move to run through the same self-check filter everything else goes through."

---

# 23. Implementing Special Move Rules

```java
public class SpecialMoveRules {
    public Optional<Move> tryKingsideCastle(Board board, GameState state, Color color) {
        if (!state.castlingRights().kingside(color)) return Optional.empty();       // king or rook has moved
        Square kingSquare = board.findKing(color);
        Square rookSquare = kingSquare.offsetBy(3, 0);
        if (board.pieceAt(rookSquare) == null) return Optional.empty();             // rook missing/moved away
        Square passThrough = kingSquare.offsetBy(1, 0);
        Square destination = kingSquare.offsetBy(2, 0);
        if (board.pieceAt(passThrough) != null || board.pieceAt(destination) != null) return Optional.empty();
        Color opponent = color.opposite();
        if (checkDetector.isSquareAttacked(board, kingSquare, opponent)
         || checkDetector.isSquareAttacked(board, passThrough, opponent)
         || checkDetector.isSquareAttacked(board, destination, opponent)) {
            return Optional.empty();  // cannot castle out of, through, or into check
        }
        return Optional.of(Move.castleKingside(color));
    }

    public Optional<Move> tryEnPassant(Board board, GameState state, Square from, Piece pawn) {
        Square target = state.enPassantTarget();  // set for exactly one turn by a two-square pawn advance -- §18
        if (target == null) return Optional.empty();
        int forward = pawn.color() == Color.WHITE ? 1 : -1;
        boolean adjacentDiagonal = from.offsetBy(-1, forward).equals(target) || from.offsetBy(1, forward).equals(target);
        return adjacentDiagonal ? Optional.of(Move.enPassantCapture(from, target, pawn)) : Optional.empty();
    }
}
```

The "cannot castle through check" rule is enforced by calling the exact same `CheckDetector.isSquareAttacked` used for ordinary check detection (§20) — not a bespoke castling-specific attack check. En passant's precondition is entirely captured by `GameState.enPassantTarget()`, which is set by a two-square pawn advance (§18) and cleared at the start of every subsequent turn — encoding "only capturable immediately" as a piece of state with a one-turn lifetime, rather than as a special rule buried in move validation. Pawn promotion is handled similarly: when a pawn's move reaches the back rank, `Move` carries an optional `promotesTo` field, and the move executor (§28) swaps the piece type at apply-time — no separate "promotion move type" branching through the rest of the pipeline.

---

# 24. Follow-up Question 7: A Piece Can Reach a Square by Its Movement Pattern, But Moving It There Exposes the King to Check — How Do You Catch This?

> **Interviewer:** *"A bishop is pinned in front of its own king. Your move generator says it can capture a pawn two squares away. What stops it from actually making that move?"*

This is precisely the two-phase structure flagged in §3: `pseudoLegalMoves` (§17-18) only knows about movement patterns and board occupancy — it has no concept of "my king" at all. The fix is a second, uniform filter applied after generation, for every piece, every turn, with no piece type exempted: simulate the move, ask "is my own king now in check," and discard the move if so. A pinned piece isn't a special case that needs its own detection logic — it's just a piece whose every pseudo-legal move happens to fail this one universal filter.

---

# 25. Implementing the Legal-Move Filter

```java
public class LegalMoveFilter {
    public List<Move> legalMovesFor(Board board, GameState state, Color color) {
        List<Move> pseudoLegal = moveGenerator.allPseudoLegalMovesFor(board, state, color);
        List<Move> legal = new ArrayList<>();
        for (Move move : pseudoLegal) {
            Board simulated = board.copyAndApply(move);         // apply on a scratch copy, never the real board
            if (!checkDetector.isInCheck(simulated, color)) {
                legal.add(move);
            }
        }
        return legal;
    }
}
```

The cost of this approach is real — every candidate move gets a full board copy and a fresh check-detection pass — but the correctness guarantee is total: there is exactly one place in the entire system where "would this expose my king" is decided, and every piece, every move type, and every caller (human move validation, AI search) goes through it identically. A performance-focused engine optimizes this loop's constant factor (incremental board updates instead of full copies, precomputed pin detection); it does not add a second code path that skips the filter.

---

# 26. Class Diagram: The Move Validation Flow

```text
┌────────────────────┐      ┌──────────────────────┐      ┌────────────────────┐
│   MoveGenerator       │─────▶│  pseudoLegalMoves()     │─────▶│  LegalMoveFilter      │
│  ───────────────────│      │  ────────────────────│      │ ───────────────────│
│  allPseudoLegal-       │      │  (per PieceMoveStrategy)│      │ legalMovesFor()        │
│    MovesFor()          │      └──────────────────────┘      │  -> simulate + check     │
└────────────────────┘                                        └──────────┬─────────┘
                                                                            │
                              ┌──────────────────────┐                    │
                              │   SpecialMoveRules      │◀───────────────────┘
                              │  ────────────────────│  candidate castle/en passant
                              │ tryKingsideCastle()      │  moves feed into the same filter
                              │ tryEnPassant()            │
                              └──────────────────────┘
```

---

# 27. Follow-up Question 8: How Do You Support Undo and Redo Cleanly, Including for a Capture or a Castle?

> **Interviewer:** *"Undo a normal move is easy — put the piece back. Undo a capture, or a castle, or an en passant? What's your general approach?"*

A move-specific "reverse the move" function, hand-written per move type, is exactly the kind of duplicated-special-casing this guide has been avoiding everywhere else. The Command pattern gives a uniform alternative: every move, regardless of type, becomes a `MoveCommand` that knows how to `execute()` and `undo()` itself, capturing whatever state it needs to reverse cleanly at creation time — before the move is applied, while that information is still cheaply available.

---

# 28. Move History as the Command Pattern

```java
public interface MoveCommand {
    void execute(Board board, GameState state);
    void undo(Board board, GameState state);
}

public class StandardMoveCommand implements MoveCommand {
    private final Square from, to;
    private final Piece movedPiece;
    private Piece capturedPiece;         // captured lazily at execute() time, needed for undo
    private CastlingRights priorRights;  // snapshot, since a rook/king move revokes rights permanently
    private Square priorEnPassantTarget;

    public void execute(Board board, GameState state) {
        priorRights = state.castlingRights().copy();
        priorEnPassantTarget = state.enPassantTarget();
        capturedPiece = board.pieceAt(to);
        board.movePiece(from, to);
        state.updateRightsAfterMove(movedPiece, from);
        state.setEnPassantTarget(null);
    }

    public void undo(Board board, GameState state) {
        board.movePiece(to, from);
        if (capturedPiece != null) board.placePiece(to, capturedPiece);
        state.restoreCastlingRights(priorRights);
        state.setEnPassantTarget(priorEnPassantTarget);
    }
}
```

Castling and en passant get their own `MoveCommand` implementations (`CastleCommand`, `EnPassantCommand`) that each know their own extra bookkeeping — a castle also moves the rook and must undo the rook's move too; an en passant capture removes a piece from a square *other than* the destination square, which must be restored to exactly that square, not to the destination, on undo. Each implementation is small and self-contained precisely because the interface forces "how do I reverse this" to be answered once, per move type, at the type's own definition — not centrally, by a function that has to know about every move type's quirks at once.

---

# 29. Implementing Undo/Redo via Move History

```java
public class MoveHistory {
    private final List<MoveCommand> executed = new ArrayList<>();
    private int cursor = 0;  // index of the next move to redo, if any

    public void recordAndExecute(MoveCommand command, Board board, GameState state) {
        command.execute(board, state);
        // discard any redo-able tail -- a new move after undoing invalidates the old future
        while (executed.size() > cursor) executed.remove(executed.size() - 1);
        executed.add(command);
        cursor++;
    }

    public boolean undo(Board board, GameState state) {
        if (cursor == 0) return false;
        cursor--;
        executed.get(cursor).undo(board, state);
        return true;
    }

    public boolean redo(Board board, GameState state) {
        if (cursor >= executed.size()) return false;
        executed.get(cursor).execute(board, state);
        cursor++;
        return true;
    }
}
```

The `cursor` — rather than always popping from the end of the list — is what makes redo possible after an undo: the history array keeps the discarded-but-not-yet-overwritten moves around until a genuinely new move is played, at which point (per the standard undo/redo convention) the old redo-able future is discarded, since it no longer corresponds to a coherent continuation of the game.

---

# 30. Follow-up Question 9: How Do You Detect Threefold Repetition and the Fifty-Move Rule Without Re-Scanning the Entire Game History on Every Move?

> **Interviewer:** *"Threefold repetition needs you to compare the current position against every prior position. Doesn't that get expensive as the game goes on?"*

Re-comparing the full board against every one of potentially hundreds of prior positions on every single move is exactly the kind of "recompute everything from scratch on every event" mistake this series has flagged before (BookMyShow's seat map, the fantasy-sports leaderboard) — the fix here is the same shape: maintain a running, incrementally-updated index rather than a full history rescan.

---

# 31. Implementing Draw Condition Detection

```java
public class DrawConditionDetector {
    private final Map<Long, Integer> positionHashCounts = new HashMap<>();  // Zobrist-style hash -> occurrences

    public void recordPosition(long positionHash) {
        positionHashCounts.merge(positionHash, 1, Integer::sum);
    }

    public void unrecordPosition(long positionHash) {  // called on undo -- §29
        positionHashCounts.computeIfPresent(positionHash, (h, count) -> count > 1 ? count - 1 : null);
    }

    public Optional<GameStatus> checkDrawConditions(GameState state) {
        if (positionHashCounts.getOrDefault(state.currentPositionHash(), 0) >= 3) {
            return Optional.of(GameStatus.THREEFOLD_REPETITION);
        }
        if (state.halfmoveClock() >= 100) {  // 50 full moves = 100 halfmoves, no capture or pawn move
            return Optional.of(GameStatus.FIFTY_MOVE_RULE);
        }
        if (hasInsufficientMaterial(state.board())) {
            return Optional.of(GameStatus.INSUFFICIENT_MATERIAL);
        }
        return Optional.empty();
    }
}
```

A position's hash is computed incrementally as part of every move's `execute()`/`undo()` (an XOR-based Zobrist hash updates in constant time per piece movement, rather than re-hashing the whole board), so `positionHashCounts` stays a small, O(1)-updated map instead of a growing list that needs re-scanning — the count lookup for repetition is then just one map read, regardless of how long the game has run.

---

# 32. Follow-up Question 10: How Do You Build an AI Opponent That's Neither Trivially Bad Nor Unacceptably Slow?

> **Interviewer:** *"Your AI needs to pick a move. What's the algorithm, and how do you keep it fast enough to feel responsive?"*

A move-quality AI needs to look ahead — evaluating only the immediate result of each candidate move (a purely greedy, one-ply choice) plays badly, because it can't see a move that looks fine now but loses material two moves later. Looking ahead multiple plies, for both sides alternately, and assuming each side plays its own best available response, is exactly what minimax formalizes; the "unacceptably slow" half of the question is answered not by looking *less* far ahead, but by pruning the search so it explores far fewer branches while still guaranteeing the identical answer minimax would have found — which is what alpha-beta pruning does.

---

# 33. The Minimax Algorithm for Move Selection

```text
Minimax builds a game tree: at each ply, the side to move picks the move that is
best FOR THEM -- meaning the maximizing side picks the highest-evaluated child,
and the minimizing side picks the lowest-evaluated child, all the way down to a
fixed search depth, where a static evaluation function (§36-37) scores the leaf
position numerically.

                    (White to move, depth 3)
                            │
              ┌─────────────┼─────────────┐
           move A         move B         move C          <- White picks MAX of these
              │             │              │
        ┌─────┼─────┐  ┌───┼───┐    ┌─────┼─────┐
      a1     a2    a3  b1  b2  b3   c1    c2    c3       <- Black picks MIN of these
        │      │     │   │    │      │      │     │
      eval  eval  eval eval eval  eval  eval  eval       <- leaf: static evaluation (§36-37)

White ultimately chooses whichever of {move A, move B, move C} has the highest
value AFTER Black has already chosen the worst-for-White response beneath it --
this is what "assume the opponent plays their best move too" means concretely.
```

---

# 34. Alpha-Beta Pruning

Minimax as described explores every branch of the tree, even ones that could never affect the final decision. Alpha-beta pruning tracks two running bounds while searching — `alpha` (the best value the maximizing side can already guarantee elsewhere) and `beta` (the best value the minimizing side can already guarantee elsewhere) — and stops exploring a branch the instant it proves that branch cannot possibly change the outcome, because the opponent would never let the game reach it.

```text
If, while searching Black's responses to White's move B, we find a response b1
that's already WORSE for White than move A's guaranteed value -- White will simply
never play move B, since move A is already known to be at least as good.
There is no need to evaluate b2 or b3 at all: whatever they turn out to be,
they can only make move B look even worse (or unchanged) for White, and White
had already rejected the earlier options based on the information available.
This is a PRUNE, not an approximation -- alpha-beta always returns the exact
same move minimax would have chosen, just without wasting time proving what
doesn't need proving.
```

---

# 35. Implementing an AI Player with Alpha-Beta

```java
public class AlphaBetaSearch {
    public Move findBestMove(GameState state, int depth) {
        Move best = null;
        int bestValue = Integer.MIN_VALUE;
        for (Move move : legalMoveFilter.legalMovesFor(state.board(), state, state.sideToMove())) {
            GameState next = state.copyAndApply(move);   // same LegalMoveFilter every human move uses -- §25
            int value = -alphaBeta(next, depth - 1, Integer.MIN_VALUE, Integer.MAX_VALUE);
            if (value > bestValue) { bestValue = value; best = move; }
        }
        return best;
    }

    private int alphaBeta(GameState state, int depth, int alpha, int beta) {
        if (depth == 0 || state.computeStatus(checkDetector, drawDetector) != GameStatus.IN_PROGRESS) {
            return boardEvaluator.evaluate(state.board(), state.sideToMove());  // §36-37
        }
        int value = Integer.MIN_VALUE;
        for (Move move : legalMoveFilter.legalMovesFor(state.board(), state, state.sideToMove())) {
            GameState next = state.copyAndApply(move);
            value = Math.max(value, -alphaBeta(next, depth - 1, -beta, -alpha));
            alpha = Math.max(alpha, value);
            if (alpha >= beta) break;  // prune -- the opponent will never let this branch be reached
        }
        return value;
    }
}
```

This is the "negamax" formulation (each recursive call negates and swaps the roles of maximizing/minimizing) — a standard, purely mechanical simplification of writing minimax and alpha-beta as one function instead of two mirror-image ones, not a different algorithm. Note the call to `legalMoveFilter.legalMovesFor` — the AI's search tree is built entirely out of moves that already passed the exact same self-check filter (§25) a human's move would have to pass; there's no separate, potentially-inconsistent "AI move generation" path.

---

# 36. Follow-up Question 11: How Do You Turn a Board Position Into a Single Number the Search Can Compare?

> **Interviewer:** *"Minimax needs a numeric score at the leaves. How do you compute that?"*

A static evaluation function has to compress "how good is this position" into one number, using signals that are cheap to compute (since it runs at every leaf of a potentially large search tree) but still meaningfully correlated with a strong position — material balance is the obvious first signal, but a competent evaluator layers in positional signals too, since material-equal positions are very often not equally good.

---

# 37. Implementing Board Evaluation

```java
public class BoardEvaluator {
    private static final Map<PieceType, Integer> MATERIAL_VALUE = Map.of(
        PieceType.PAWN, 100, PieceType.KNIGHT, 320, PieceType.BISHOP, 330,
        PieceType.ROOK, 500, PieceType.QUEEN, 900, PieceType.KING, 0
    );

    public int evaluate(Board board, Color perspective) {
        int score = 0;
        for (Square square : board.occupiedSquares()) {
            Piece piece = board.pieceAt(square);
            int materialValue = MATERIAL_VALUE.get(piece.type());
            int positionalValue = pieceSquareTable(piece.type())[square.index()];  // e.g. knights score higher centralized
            int contribution = materialValue + positionalValue;
            score += (piece.color() == perspective) ? contribution : -contribution;
        }
        return score;
    }
}
```

A piece-square table is just a precomputed, per-piece-type array of small positional bonuses/penalties indexed by square — a knight on the rim is worth less than one centralized, a king is safer castled behind pawns in the midgame but wants to be active in the endgame. None of this changes the search algorithm in §35 at all; it only changes what number comes back at the leaves, which is precisely the separation of concerns that lets the evaluation function be tuned or replaced (even by a trained model) without touching the search.

---

# 38. Follow-up Question 12: Two Remote Players Are in the Same Game — What Stops Both From Submitting a Move at Once, Out of Turn?

> **Interviewer:** *"Both players' clients think it's their turn due to a UI bug, or one player double-clicks submit — what actually enforces turn order?"*

This is the same structural problem this series has solved for a contest slot, a wallet balance, and cash in a dispenser: a naive "check whose turn it is, then apply the move" is a check-then-act race. The fix is the identical atomic-conditional-update pattern, applied here to a game's move counter instead of a slot count or a balance.

---

# 39. Implementing Concurrency-Safe Move Submission

```java
public class ConcurrentMoveSubmission {
    public MoveResult submitMove(String gameId, Color submittingColor, Move move, int expectedMoveNumber) {
        int rowsUpdated = jdbcTemplate.update(
            "UPDATE game SET move_number = move_number + 1 " +
            "WHERE game_id = ? AND side_to_move = ? AND move_number = ?",
            gameId, submittingColor.name(), expectedMoveNumber
        );
        if (rowsUpdated == 0) {
            return MoveResult.rejected("Not your turn, or this move was already superseded");
        }
        // rowsUpdated == 1: this request atomically won the right to apply this move number
        applyMoveAndPersist(gameId, move);
        return MoveResult.accepted();
    }
}
```

The `WHERE side_to_move = ? AND move_number = ?` guard is evaluated by the database against the *current* row in one atomic statement — exactly BookMyShow's `SELECT ... FOR UPDATE`, the ATM's `AtomicInteger.updateAndGet`, and the fantasy-sports guide's `WHERE filled_slots < total_slots`, now guarding "whose turn is it, and which move number are we on" instead of a seat, a balance, or a contest slot. Two near-simultaneous submissions for the same move number can never both succeed — the second one's `UPDATE` simply affects zero rows once the first has already advanced `move_number`.

---

# 40. Class Diagram: The Full Chess Engine

```text
┌───────────────┐     ┌───────────────┐     ┌────────────────────┐
│  GameController │────▶│  GameState      │────▶│  LegalMoveFilter      │
│  (§39 guard)     │     │  (§13-14)       │     │  (§25)                │
└───────────────┘     └───────┬───────┘     └──────────┬─────────┘
                                │                            │
                                ▼                            ▼
                        ┌───────────────┐            ┌────────────────┐
                        │  MoveHistory     │            │  MoveGenerator    │
                        │  (§27-29)         │            │  (§17-18)          │
                        └───────────────┘            └────────────────┘
                                                                │
                                                                ▼
                                                        ┌────────────────┐
                                                        │  SpecialMoveRules │
                                                        │  (§22-23)          │
                                                        └────────────────┘

┌───────────────┐     ┌───────────────┐     ┌────────────────────┐
│ AlphaBetaSearch  │────▶│ BoardEvaluator   │     │ DrawConditionDetector │
│ (§34-35)          │     │ (§36-37)          │     │ (§30-31)               │
└───────────────┘     └───────────────┘     └────────────────────┘
        │ calls the SAME LegalMoveFilter + MoveGenerator as a human move
        └──────────────────────────────────────────────────────────────▶
```

---

# 41. Capacity Estimation: Search Space and Response Latency

A typical mid-game chess position has roughly 30-40 legal moves. A naive minimax search to depth 4 (two full moves ahead for each side) explores on the order of 35⁴ ≈ 1.5 million leaf positions. Alpha-beta pruning, with reasonable move ordering (trying likely-strong moves first so more branches get pruned), typically reduces this to roughly the square root of the unpruned count in the best case — closer to 35² ≈ 1,225 leaf evaluations for the same depth, a few orders of magnitude fewer positions to statically evaluate. This is the concrete, numeric reason alpha-beta isn't an optional micro-optimization: it's what makes a depth-4 or depth-6 search feasible within an interactive time budget at all, on ordinary hardware, without changing the move chosen.

---

# 42. Full Worked Example: One Game Traced Through Several Moves

1. **Opening move** — White plays `e2-e4`, a two-square pawn advance; `GameState.enPassantTarget` is set to `e3` for exactly one turn (§18).
2. **Black's reply** — Black plays `e7-e5`. En passant target resets (Black's own two-square advance now sets a *new* target at `e6`, replacing the stale one).
3. **A pin appears** — several moves later, White's bishop on `b5` pins Black's knight on `c6` against Black's king on `e8`. The knight's `pseudoLegalMoves` still lists a move off the pin line, but `LegalMoveFilter` (§25) simulates it, finds Black's own king would be in check afterward, and excludes it — Black's client never even offers that square as a legal destination.
4. **Castling** — White, having moved neither king nor rook, and with `f1`/`g1` empty and unattacked, plays kingside castle; `SpecialMoveRules.tryKingsideCastle` (§23) confirms all four preconditions and produces a `CastleCommand` that moves both the king and rook in one recorded, reversible action.
5. **A missed check** — Black's move exposes their own king to a rook's attack along an open file; `CheckDetector.isInCheck` (§20) catches this on the very next `computeStatus` call, and the game state transitions to `IN_CHECK`.
6. **Checkmate is tested every turn, not declared manually** — several moves later, Black has no legal move that escapes check; `resolveTerminalStatus` (§21) returns `CHECKMATE` because `legalMovesFor` returns empty while `isInCheck` is true — nothing in the engine ever explicitly says "this is checkmate" as a special case.
7. **Undo, for review** — a spectator rewinds three moves via `MoveHistory.undo()` (§29); the pin, the castle, and the check all correctly unwind in reverse, including the rook returning from its castled square and the exposed-king detection reverting because `CheckDetector` re-evaluates fresh against the restored board — there is no separate "undo the check flag" step, because there never was a check flag to begin with, only a computation.

---

# 43. Final Architecture Diagram

```text
┌────────────┐   submit Move    ┌──────────────────┐
│ Client (UI) │─────────────────▶│  GameController     │
└────────────┘                  │  (concurrency guard,  │
       ▲                        │   §38-39)              │
       │ board state, status     └─────────┬──────────┘
       │                                   │
       │                          ┌────────▼─────────┐
       │                          │    GameState        │
       │                          │  (§13-14)            │
       │                          └────────┬─────────┘
       │              ┌────────────────────┼────────────────────┐
       │              ▼                    ▼                    ▼
       │      ┌───────────────┐   ┌────────────────┐   ┌──────────────────┐
       │      │ MoveGenerator    │   │ CheckDetector      │   │ DrawConditionDet.   │
       │      │ + SpecialMoveRules│   │ (§20-21)            │   │ (§30-31)             │
       │      │ (§17-18, §22-23)   │   └────────────────┘   └──────────────────┘
       │      └───────┬───────┘
       │              ▼
       │      ┌───────────────┐
       │      │ LegalMoveFilter  │  (§25)
       │      └───────┬───────┘
       │              ▼
       │      ┌───────────────┐        ┌────────────────────┐
       └──────│  MoveHistory     │◀───────│  AlphaBetaSearch      │
              │  (undo/redo,      │        │  + BoardEvaluator     │
              │   §27-29)          │        │  (§33-37, AI opponent)│
              └───────────────┘        └────────────────────┘
```

---

# 44. Design Patterns Used Throughout This Guide

- **Strategy** — `PieceMoveStrategy` (§17-18) isolates each piece type's movement rule behind one shared interface, with no piece type aware of any other's implementation.
- **State** — `GameStatus` (§13-14) makes the reachable combinations of check/checkmate/stalemate/draw the only representable ones.
- **Command** — `MoveCommand` (§28-29) makes every move type — standard, castle, en passant, promotion — independently executable and reversible, which is what undo/redo is built on.
- **Template Method** (implicit) — `LegalMoveFilter.legalMovesFor` (§25) is a fixed two-phase skeleton (generate pseudo-legal, then filter by simulated self-check) that every piece type and every move source (human, AI) shares without modification.
- **Facade** — `GameController` (§39) presents "submit this move" as one call while coordinating the concurrency guard, move validation, execution, and history recording underneath.

---

# 45. SOLID Principles Applied

- **Single Responsibility** — `CheckDetector` only answers "is this square/king attacked"; it has no opinion about castling rights, draw conditions, or move history.
- **Open/Closed** — adding a new piece variant (say, a custom-variant "archbishop") means writing one new `PieceMoveStrategy`; nothing in `LegalMoveFilter`, `CheckDetector`, or the AI search needs to change.
- **Liskov Substitution** — any `PieceMoveStrategy` can be swapped into `MoveGenerator` without the caller needing to know which concrete piece type it's generating moves for.
- **Interface Segregation** — `MoveCommand` exposes only `execute()`/`undo()`; it doesn't force every move type to also implement unrelated rendering or notation-export concerns.
- **Dependency Inversion** — `AlphaBetaSearch` depends on `LegalMoveFilter` and `BoardEvaluator` as collaborators, not concrete implementations, so the evaluation function (or even the search algorithm's move-ordering heuristic) can be swapped without touching the search's control flow.

---

# 46. Common Mistakes When Building This Yourself

- **Folding self-check exclusion into each piece's move generator** instead of keeping pseudo-legal generation and the legality filter as two separate phases (§17-18 vs. §25) — this is the single most common source of the classic "moved into check" or "didn't catch a pin" bug.
- **Tracking `isCheck`/`isCheckmate` as separately-maintained boolean fields** instead of computing them fresh from move generation on every turn (§13-14, §21) — a forgotten update path is how a stale flag silently disagrees with the actual board.
- **Special-casing castling and en passant throughout the general move-validation pipeline** instead of modeling them as their own move types with self-contained preconditions that still flow through the same universal legality filter (§22-23).
- **Re-scanning full board history for threefold repetition on every move** instead of maintaining an incrementally-updated position-hash count (§30-31).
- **Writing a bespoke "undo this specific move type" function per move type in one central place** instead of using the Command pattern so each move type owns its own reversal logic (§27-29) — the central-function approach is exactly the kind of tangled conditional pile this guide has avoided everywhere else.
- **Giving the AI its own, separate move-generation implementation** for search-tree expansion, "for performance," instead of reusing the identical `LegalMoveFilter` a human move goes through (§35) — a divergent AI-only move generator is how an engine ends up occasionally proposing a move a human client would have rejected as illegal.
- **Treating "submit a move" as a single, unguarded write** instead of recognizing the same turn-order race this series has repeatedly flagged for shared resources, and closing it with an atomic conditional update (§38-39).

---

# 47. Testing Strategy

- **Piece movement tests** — for each `PieceMoveStrategy`, assert the exact set of pseudo-legal destination squares from a range of board positions, including edge-of-board and fully-blocked cases.
- **Self-check filter tests** — construct positions with a pinned piece and assert its illegal-if-exposing moves are excluded while its legal-along-the-pin-line moves are retained.
- **Checkmate/stalemate tests** — use well-known checkmate and stalemate positions (e.g., the "smothered mate," a basic king-and-queen stalemate trap) and assert `computeStatus` returns the correct terminal state.
- **Special move tests** — assert castling is correctly denied when the king has moved, when a passed-through square is attacked, and when a blocking piece is present; assert en passant is only available exactly one turn after the qualifying two-square pawn advance, never before or after.
- **Undo/redo tests** — execute a sequence including a capture, a castle, and an en passant, undo all of them, and assert the board and game state are byte-for-byte identical to the pre-sequence snapshot; then redo and assert the post-sequence state is restored identically.
- **Draw condition tests** — construct a position repeated three times and assert threefold repetition triggers; construct 100 halfmoves with no capture or pawn move and assert the fifty-move rule triggers.
- **Alpha-beta correctness tests** — run plain minimax and alpha-beta on the same position to the same depth and assert they select the identical move, across a range of positions — this is the concrete way to verify pruning changed performance, not the answer.
- **Concurrent move submission tests** — fire two simultaneous move submissions for the same game and move number, and assert exactly one is accepted.

---

# 48. Suggested Future Enhancements

- **Bitboard representation** — replacing the object-array board with 64-bit integer bitboards for dramatically faster move generation, which the AI search benefits from directly without any change to the search algorithm itself (§33-35).
- **Opening book and endgame tablebases** — short-circuiting search entirely for well-known early and late-game positions, bounding how deep the AI ever actually needs to search in practice.
- **Iterative deepening with a time budget** — running alpha-beta at increasing depths until a wall-clock budget is nearly spent, returning the best move found at the deepest fully-completed depth, which decouples "how strong is the AI" from "how long will this position take."
- **Chess variants** — Chess960 (randomized starting position) reuses every piece of this design unchanged except the initial board setup and a small adjustment to castling's precondition checks (§22-23).
- **Spectator mode and live move broadcast** — reusing the same WebSocket transport already needed for two-player multiplayer (§4), fanning move updates out to any number of read-only observers.

---

# 49. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Design move generation so that adding a new piece type never requires touching existing piece logic. (§16-18)
2. A piece can reach a square by its movement pattern, but the move would expose the mover's own king — design the fix, and explain why it must be a separate phase. (§24-25)
3. Detect checkmate and stalemate correctly, and explain why they should not be two independently-implemented algorithms. (§19-21)
4. Design castling and en passant so neither one leaks special-case logic into the general move-validation pipeline. (§22-23)
5. Design undo/redo so it correctly handles a capture, a castle, and an en passant — not just a simple, reversible quiet move. (§27-29)
6. Design threefold-repetition detection that doesn't re-scan the entire game history on every move. (§30-31)
7. Design an AI opponent using minimax, then explain precisely why alpha-beta pruning returns the identical move while exploring far fewer positions. (§32-35)
8. Two remote players' clients both believe it's their turn — design the fix, precisely. (§38-39)

---

# 50. Final Takeaway

Nearly every hard rule in chess reduces to one recurring discipline: **compute derived facts fresh from first principles, and reuse the same underlying computation everywhere it's needed, rather than maintaining a separate, parallel answer that can silently drift out of sync.** Check, checkmate, and stalemate are not three algorithms — they are two computations (attack detection, legal-move generation) combined two different ways (§19-21). Castling and en passant are not special rules bolted onto move validation — they are ordinary candidate moves with their own preconditions, filtered by the exact same self-check filter every other move goes through (§22-25). Even the AI opponent follows this discipline: it searches using the identical legal-move generator a human move must pass, meaning "make the engine smarter" and "make the engine follow the rules" can never accidentally trade off against each other (§35). Recognizing which facts in a system should be *computed on demand* rather than *separately tracked* — and having the discipline to route every caller through one shared computation instead of a convenient local shortcut — is the transferable skill this guide is really teaching.

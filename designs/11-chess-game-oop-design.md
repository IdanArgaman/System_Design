# Chess Game — OOP Design for an Interview

## 1. Goal

Design a chess game from an OOP perspective with:

- Clean domain modeling
- SOLID principles
- Separation of responsibilities
- Polymorphic piece movement
- Clear handling of legal moves
- Detailed handling of check, checkmate, and stalemate

The goal is **design clarity**, not an exact production implementation.

---

# 2. High-Level Architecture

```text
                         ┌──────────────┐
                         │     Game     │
                         └──────┬───────┘
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
             ┌───────┐      ┌────────┐   ┌──────────┐
             │ Board │      │ Player │   │   Move   │
             └───┬───┘      └────────┘   └──────────┘
                 │
                 ▼
        ┌─────────────────┐
        │      Piece      │
        └────────┬────────┘
                 │
      ┌──────────┼───────────┬─────────┐
      ▼          ▼           ▼         ▼
    King       Queen        Rook     Knight
    Bishop     Pawn         ...

                         Rules
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌────────────┐ ┌────────────┐ ┌──────────────┐
       │MoveValidator│ │CheckDetector│ │MoveGenerator│
       └────────────┘ └────────────┘ └──────────────┘
              │            │            │
              └────────────┼────────────┘
                           ▼
                  ┌───────────────────┐
                  │GameStateEvaluator │
                  └───────────────────┘
```

---

# 3. Core Domain Objects

## Color

```ts
enum Color {
  WHITE,
  BLACK
}
```

---

## Position

A position represents one square on the board.

```ts
class Position {
  constructor(
    readonly row: number,
    readonly column: number
  ) {}

  equals(other: Position): boolean {
    return this.row === other.row &&
           this.column === other.column;
  }

  isValid(): boolean {
    return (
      this.row >= 0 &&
      this.row < 8 &&
      this.column >= 0 &&
      this.column < 8
    );
  }
}
```

Using a `Position` object is preferable to passing strings such as `"e4"` throughout the domain.

---

# 4. Board

The `Board` is responsible for **board state**.

It should not be responsible for determining check, checkmate, or whether a move is legal.

```ts
class Board {
  private pieces: Map<string, Piece>;

  getPiece(position: Position): Piece | null;

  placePiece(piece: Piece, position: Position): void;

  removePiece(position: Position): void;

  movePiece(from: Position, to: Position): void;

  getPieces(color?: Color): Piece[];

  clone(): Board;
}
```

### Board responsibilities

- Store pieces
- Find pieces
- Add/remove pieces
- Move pieces
- Provide board state
- Create a copy for move simulation

### Board should NOT do

```text
board.isCheck(...)
board.isCheckmate(...)
board.isLegalMove(...)
```

Those belong to the rules/services layer.

---

# 5. Piece Hierarchy

Use inheritance + polymorphism for chess pieces.

```text
             Piece
               │
       ┌───────┼────────┐
       │       │        │
     King    Queen     Rook
       │       │        │
    Bishop  Knight     Pawn
```

Base class:

```ts
abstract class Piece {
  constructor(
    readonly color: Color,
    protected position: Position
  ) {}

  abstract getPossibleMoves(board: Board): Position[];

  abstract attacks(
    target: Position,
    board: Board
  ): boolean;

  getPosition(): Position {
    return this.position;
  }
}
```

Concrete pieces:

```ts
class King extends Piece {
  getPossibleMoves(board: Board): Position[] {
    // One square in any direction.
  }

  attacks(target: Position, board: Board): boolean {
    // King attack pattern.
  }
}
```

```ts
class Knight extends Piece {
  getPossibleMoves(board: Board): Position[] {
    // L-shaped movement.
  }

  attacks(target: Position, board: Board): boolean {
    // Knight attack pattern.
  }
}
```

```ts
class Rook extends Piece {
  getPossibleMoves(board: Board): Position[] {
    // Horizontal / vertical movement.
  }

  attacks(target: Position, board: Board): boolean {
    // Straight-line attack if path is clear.
  }
}
```

Similar implementations exist for:

- Queen
- Bishop
- Pawn

---

# 6. Important Distinction: Possible vs Legal Moves

This is one of the most important design concepts.

## Pseudo-legal / possible move

A move that follows the movement rules of the piece.

Example:

```ts
piece.getPossibleMoves(board)
```

For example, a rook can theoretically move vertically or horizontally.

But this doesn't mean the move is actually legal.

---

## Legal move

A move that:

1. Follows the piece's movement rules
2. Doesn't land on a square occupied by a friendly piece
3. Respects other chess rules
4. Does **not leave the player's own king in check**

Therefore:

```text
Piece movement
       ↓
Possible moves
       ↓
Move validation
       ↓
Legal moves
```

This separation is essential for a clean design.

---

# 7. Move

Represent a move as a domain object.

```ts
class Move {
  constructor(
    readonly piece: Piece,
    readonly from: Position,
    readonly to: Position,
    readonly capturedPiece?: Piece
  ) {}
}
```

A move can later contain additional information for special moves:

```text
Move
 ├── normal move
 ├── capture
 ├── castling
 ├── promotion
 └── en passant
```

The important idea is that the rest of the system can work with a `Move` rather than passing around multiple independent parameters.

---

# 8. MoveValidator

The `MoveValidator` answers:

> "Can this move legally be played?"

```ts
class MoveValidator {
  constructor(
    private readonly checkDetector: CheckDetector
  ) {}

  isLegal(game: Game, move: Move): boolean {
    // 1. Correct player's turn
    // 2. Piece belongs to current player
    // 3. Piece can move to destination
    // 4. Destination isn't occupied by own piece
    // 5. Simulate the move
    // 6. Verify own king isn't in check
  }
}
```

The validator is deliberately separate from `Piece`.

A `Piece` should not need to know about:

- Whose turn it is
- The entire game state
- Checkmate
- Other players
- Game history

---

# 9. Check Detection

The fundamental operation is:

```ts
isInCheck(board: Board, color: Color): boolean
```

The algorithm:

### Step 1 — Find the player's king

```ts
const king = board
  .getPieces(color)
  .find(piece => piece instanceof King);
```

### Step 2 — Find all opponent pieces

```ts
const opponentPieces =
  board.getPieces(opponent(color));
```

### Step 3 — Check whether any opponent piece attacks the king

```ts
for (const piece of opponentPieces) {
  if (
    piece.attacks(
      king.getPosition(),
      board
    )
  ) {
    return true;
  }
}

return false;
```

Conceptually:

```text
                 Player King
                     ↓
                 King position
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
   Enemy Rook    Enemy Bishop   Enemy Knight
       │             │             │
       └─────────────┼─────────────┘
                     ↓
             Does anything attack
                  the King?
                 /         \
               YES          NO
                │            │
              CHECK        SAFE
```

---

# 10. Why Attack Detection Is Separate

There is a subtle but important difference between:

```text
legal moves
```

and:

```text
squares a piece attacks
```

For example, kings cannot legally move next to each other.

However, each king still attacks the squares immediately surrounding it.

Therefore, check detection should conceptually ask:

> "Does this enemy piece attack the king's square?"

rather than:

> "Does this enemy piece have a legal move to the king's square?"

A useful abstraction is:

```ts
abstract class Piece {
  abstract attacks(
    target: Position,
    board: Board
  ): boolean;
}
```

Then each piece knows its attack pattern.

This avoids a large `switch` statement such as:

```ts
switch (piece.type) {
  case KING:
  case QUEEN:
  case ROOK:
  ...
}
```

and makes the design more polymorphic.

---

# 11. CheckDetector

A dedicated service can coordinate the attack detection:

```ts
class CheckDetector {
  isInCheck(
    board: Board,
    color: Color
  ): boolean {
    const king = this.findKing(board, color);

    const opponentPieces =
      board.getPieces(opponent(color));

    return opponentPieces.some(piece =>
      piece.attacks(
        king.getPosition(),
        board
      )
    );
  }

  private findKing(
    board: Board,
    color: Color
  ): King {
    // Find and return the king.
  }
}
```

Responsibility:

> Determine whether a player's king is currently under attack.

It should not determine checkmate.

---

# 12. MoveGenerator

The `MoveGenerator` is responsible for finding all legal moves.

```ts
class MoveGenerator {
  constructor(
    private readonly moveValidator: MoveValidator
  ) {}

  getLegalMoves(
    game: Game,
    color: Color
  ): Move[] {
    const legalMoves: Move[] = [];

    for (const piece of game.board.getPieces(color)) {
      const possibleMoves =
        piece.getPossibleMoves(game.board);

      for (const destination of possibleMoves) {
        const move = this.createMove(
          piece,
          destination
        );

        if (
          this.moveValidator.isLegal(
            game,
            move
          )
        ) {
          legalMoves.push(move);
        }
      }
    }

    return legalMoves;
  }
}
```

Conceptually:

```text
All pieces
    ↓
Possible moves
    ↓
Validate each move
    ↓
Legal moves
```

---

# 13. The Critical Part: Simulating a Move

How do we determine whether a move leaves our king in check?

We simulate it.

Example:

```text
Before:

Black Rook
    │
    │
White Rook
    │
White King
```

If White moves the rook away, the black rook may gain a line of attack to the white king.

Therefore, the move is illegal.

We can detect this by using a board copy:

```ts
isLegal(game: Game, move: Move): boolean {
  // Basic validation...

  const simulatedBoard =
    game.board.clone();

  simulatedBoard.movePiece(
    move.from,
    move.to
  );

  if (
    this.checkDetector.isInCheck(
      simulatedBoard,
      move.piece.color
    )
  ) {
    return false;
  }

  return true;
}
```

The flow is:

```text
             Candidate move
                   │
                   ▼
          Create board copy
                   │
                   ▼
             Apply move
                   │
                   ▼
       Is own king in check?
             /          \
           YES           NO
            │             │
         Illegal         Legal
```

This automatically handles many complicated situations, including **pinned pieces**.

---

# 14. Checkmate

Checkmate is:

```text
Player is in check
        AND
Player has no legal moves
        =
Checkmate
```

Implementation concept:

```ts
isCheckmate(
  game: Game,
  color: Color
): boolean {
  const inCheck =
    this.checkDetector.isInCheck(
      game.board,
      color
    );

  if (!inCheck) {
    return false;
  }

  const legalMoves =
    this.moveGenerator.getLegalMoves(
      game,
      color
    );

  return legalMoves.length === 0;
}
```

This is a powerful design because checkmate is built from two existing concepts:

```text
CheckDetector
      +
MoveGenerator
      ↓
Checkmate
```

---

# 15. Why the King Is Never Captured

In chess, the game doesn't actually end by capturing the king.

Instead:

```text
King is attacked
+
No legal move can remove the attack
=
Checkmate
```

Therefore, the engine should never need to perform:

```ts
captureKing();
```

The king remains on the board in the final checkmate position.

---

# 16. Stalemate

We must distinguish checkmate from stalemate.

## Checkmate

```text
In check
+
No legal moves
=
CHECKMATE
```

## Stalemate

```text
Not in check
+
No legal moves
=
STALEMATE
```

Therefore:

```ts
if (legalMoves.length === 0) {
  if (inCheck) {
    return GameStatus.CHECKMATE;
  }

  return GameStatus.STALEMATE;
}
```

---

# 17. Game State

Represent the state explicitly:

```ts
enum GameStatus {
  ACTIVE,
  CHECK,
  CHECKMATE,
  STALEMATE,
  DRAW
}
```

A dedicated evaluator can determine the state.

```ts
class GameStateEvaluator {
  constructor(
    private readonly checkDetector: CheckDetector,
    private readonly moveGenerator: MoveGenerator
  ) {}

  evaluate(game: Game): GameStatus {
    const color =
      game.currentPlayer.color;

    const inCheck =
      this.checkDetector.isInCheck(
        game.board,
        color
      );

    const legalMoves =
      this.moveGenerator.getLegalMoves(
        game,
        color
      );

    if (legalMoves.length === 0) {
      return inCheck
        ? GameStatus.CHECKMATE
        : GameStatus.STALEMATE;
    }

    return inCheck
      ? GameStatus.CHECK
      : GameStatus.ACTIVE;
  }
}
```

---

# 18. Game

The `Game` coordinates the domain.

```ts
class Game {
  constructor(
    readonly board: Board,
    readonly players: Player[],
    private readonly moveValidator: MoveValidator,
    private readonly stateEvaluator: GameStateEvaluator
  ) {}

  makeMove(move: Move): void {
    if (
      !this.moveValidator.isLegal(
        this,
        move
      )
    ) {
      throw new Error("Illegal move");
    }

    this.board.movePiece(
      move.from,
      move.to
    );

    this.switchTurn();
  }

  getStatus(): GameStatus {
    return this.stateEvaluator.evaluate(this);
  }
}
```

`Game` coordinates the process but doesn't contain the detailed chess rules.

It should not know:

```text
How a rook moves
How a knight moves
How a bishop attacks
How check is detected
How checkmate is calculated
```

Those responsibilities belong elsewhere.

---

# 19. SOLID Principles

## S — Single Responsibility Principle

Each class has a focused responsibility:

```text
Board
  → stores board state

Piece
  → knows its movement/attack behavior

Move
  → represents a move

MoveValidator
  → validates whether a move is legal

CheckDetector
  → determines whether a king is in check

MoveGenerator
  → generates legal moves

GameStateEvaluator
  → determines CHECK / CHECKMATE / STALEMATE

Game
  → coordinates the game
```

---

## O — Open/Closed Principle

Adding a new piece shouldn't require changing the entire game.

```ts
class NewPiece extends Piece {
  // movement and attack behavior
}
```

The rest of the system can continue working against:

```ts
Piece
```

rather than concrete piece types.

---

## L — Liskov Substitution Principle

Every concrete piece should be usable wherever a `Piece` is expected:

```text
Piece
 ├── King
 ├── Queen
 ├── Rook
 ├── Bishop
 ├── Knight
 └── Pawn
```

A consumer should not need to know which concrete piece it received.

---

## I — Interface Segregation Principle

Avoid a giant interface such as:

```ts
interface ChessEverything {
  move();
  capture();
  castle();
  promote();
  detectCheck();
  detectCheckmate();
}
```

Prefer focused abstractions:

```ts
interface MoveValidator {
  isLegal(...);
}

interface CheckDetector {
  isInCheck(...);
}

interface MoveGenerator {
  getLegalMoves(...);
}
```

---

## D — Dependency Inversion Principle

Prefer dependency injection:

```ts
class Game {
  constructor(
    private moveValidator: MoveValidator,
    private stateEvaluator: GameStateEvaluator
  ) {}
}
```

rather than:

```ts
class Game {
  private validator =
    new MoveValidator();

  private evaluator =
    new GameStateEvaluator();
}
```

This makes the system easier to:

- Test
- Replace implementations
- Mock dependencies
- Extend

---

# 20. The Most Important Conceptual Separation

Keep these three levels separate:

```text
┌──────────────────────────────┐
│  Piece Movement              │
│                              │
│  "Can a rook move A → B?"    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│  Move Legality               │
│                              │
│  "Can this move actually     │
│   be played in this state?"  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│  Game State                  │
│                              │
│  "Is the player in check,    │
│   checkmate or stalemate?"   │
└──────────────────────────────┘
```

This prevents a giant method such as:

```ts
ChessGame.isValidMove()
```

containing hundreds of conditional statements.

---

# 21. Interview Answer: "How Do You Determine the Winner?"

A strong concise answer:

> In chess, I don't determine the winner by capturing the king. After every move, I evaluate the current player's position.
>
> First, I find their king and determine whether it is attacked by any opponent piece. If it is, the player is in check.
>
> Then I generate all possible moves for that player and validate each one. Validation includes simulating the move and verifying that the player's own king isn't left in check.
>
> If the player is in check and has zero legal moves, it's checkmate and the opponent wins.
>
> If the player has zero legal moves but isn't in check, it's stalemate.
>
> This keeps movement rules, move legality, check detection, and game-state evaluation as separate responsibilities.

---

# 22. Special Moves — Follow-Up Topics

If the interviewer asks about the more advanced rules, the design should be extendable to:

### Castling

Requires game state such as:

- King hasn't moved
- Rook hasn't moved
- Squares between them are empty
- King isn't currently in check
- King doesn't pass through an attacked square
- King doesn't end on an attacked square

This suggests that some rules depend on **game history**, not just the current board.

---

### En Passant

Requires knowledge of the previous move.

For example:

```ts
class Game {
  private moveHistory: Move[];
}
```

The move validator can use the latest move when determining whether en passant is available.

---

### Pawn Promotion

When a pawn reaches the final rank:

```text
Pawn
  ↓
Promotion
  ↓
Queen / Rook / Bishop / Knight
```

The promotion choice can be represented as part of the `Move` or as a separate command/value object.

---

# 23. Potential Interview Trade-Off: Board Copying

The simplest correct approach is:

```text
For every candidate move:
    clone board
    apply move
    check king
```

This is excellent for clarity and an interview-level design.

For a high-performance chess engine, repeatedly copying the board can become expensive.

A production engine might instead:

```text
make move
   ↓
check
   ↓
undo move
```

using a reversible move representation.

For example:

```ts
class Move {
  from: Position;
  to: Position;
  capturedPiece?: Piece;
  previousState?: GameState;
}
```

Then:

```text
makeMove()
   ↓
evaluate
   ↓
undoMove()
```

The architectural idea remains the same; only the simulation strategy changes.

---

# 24. Final Design Summary

```text
Game
 │
 ├── Board
 │    └── Piece[]
 │
 ├── Player[]
 │
 ├── MoveValidator
 │
 └── GameStateEvaluator
       │
       ├── CheckDetector
       │
       └── MoveGenerator
              │
              └── MoveValidator

Piece
 │
 ├── King
 ├── Queen
 ├── Rook
 ├── Bishop
 ├── Knight
 └── Pawn
```

The key design principles are:

1. **Pieces own movement behavior.**
2. **Board owns state, not rules.**
3. **MoveValidator determines legal moves.**
4. **CheckDetector determines whether a king is attacked.**
5. **MoveGenerator finds all legal moves.**
6. **GameStateEvaluator determines check, checkmate, and stalemate.**
7. **Game coordinates the overall flow.**
8. **Simulate candidate moves to detect moves that expose the king.**
9. **Use polymorphism instead of large type-based conditionals.**
10. **Keep dependencies injectable to support SOLID and testing.**

The most important equation is:

```text
CHECKMATE =
    IN_CHECK
    AND
    LEGAL_MOVES == 0
```

while:

```text
STALEMATE =
    NOT_IN_CHECK
    AND
    LEGAL_MOVES == 0
```

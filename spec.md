# Logic Deduction Puzzle -- Procedural Generator Specification

## Overview

This document defines the rules, data model, constraint system, solver
behavior, and procedural generation pipeline for a **grid-based logic
deduction puzzle** inspired by games like *Clues by Sam*.\
The system is designed to be **generated and solved locally using pure
JavaScript with no external dependencies**, enabling offline execution
from a single HTML file.

The goal is to generate puzzles that: - Have **exactly one valid
solution** - Are **logically solvable without guessing** (optional
strict mode) - Use a small set of expressive clue types - Support
**procedural generation of thousands of puzzles**

------------------------------------------------------------------------

# 1. Game Structure

## Grid

-   Grid size: **5 rows × 4 columns (20 tiles total)**
-   Each tile represents a **suspect**
-   Each suspect has:
    -   a **name**
    -   a **binary state**
    -   an **occupation**
    -   a **character-appropriate emoji representing them based on their gender and occupation**

### Binary state

Each tile is one of: - **Innocent** - **Criminal**

In solver representation:

    -1 = unknown
    0 = innocent
    1 = criminal

### Occupations

Each tile has an occupation label.

Typical puzzle: - **7--10 occupation types** - Each occupation appears
**2--4 times**

Occupations create **non-spatial logical relationships** across the
board.

------------------------------------------------------------------------

# 2. Puzzle Objective

The player must determine which tiles are **criminals** and which are
**innocent** using logical deduction from the provided clues.

A puzzle is complete when the player has correctly labeled all 20 tiles.

Valid puzzles must satisfy:

-   Exactly **one solution**
-   Solvable through logical deductions
-   No guessing required (optional strict mode)

------------------------------------------------------------------------

# 3. Constraint Model

All clues are represented internally as **constraints over subsets of
tiles**.

This allows every clue type to be expressed using a small mathematical
core.

Let:

    x_i = 1 if tile i is criminal
    x_i = 0 if tile i is innocent

------------------------------------------------------------------------

# 4. Constraint Types

## 4.1 Subset Count Constraint

Form:

    k_min ≤ Σ x_i ≤ k_max

Where the sum is over a set of tiles.

Examples:

### Row constraint

"Row 2 contains exactly 3 criminals"

    S = tiles in row 2
    k_min = 3
    k_max = 3

### Region constraint

"In this group of tiles there are exactly 2 criminals"

    S = region tiles
    k_min = 2
    k_max = 2

### Neighborhood clue

"This suspect has 2 criminal neighbors"

    S = 8 surrounding tiles
    k_min = 2
    k_max = 2

------------------------------------------------------------------------

## 4.2 Comparison Constraint

Form:

    Σ(A) op Σ(B) + offset

Where:

    op ∈ { <, =, > }

### Examples

#### Row comparison

"Row 1 has more criminals than Row 3"

    Σ(Row1) > Σ(Row3)

#### Occupation comparison

"There are as many innocent judges as innocent cops"

Let:

    |J| = number of judge tiles
    |C| = number of cop tiles

Innocent equality:

    (|J| - Σ(J)) = (|C| - Σ(C))

Rewritten:

    Σ(J) = Σ(C) + (|J| - |C|)

------------------------------------------------------------------------

# 5. Clue Categories

The generator uses a limited set of clue types.

## Spatial clues

### Neighbor count

-   Counts criminals in the **8 neighboring cells**

### Row totals

-   Exact number of criminals in a row

### Column totals

-   Exact number of criminals in a column

### Region sets

-   Arbitrary subsets of tiles
-   Often 4--7 tiles

------------------------------------------------------------------------

## Global clues

### Row/column comparisons

Examples:

    Row1 > Row3
    Column2 = Column4

### Occupation relationships

Examples:

    Innocent judges = innocent cops
    Criminal doctors > criminal teachers

These clues connect tiles **across the board**, increasing puzzle
complexity.

------------------------------------------------------------------------

# 6. Solver Design

The solver performs:

1.  **Constraint propagation**
2.  **Backtracking search**
3.  **Uniqueness verification**
4.  Optional **forced-move validation**

------------------------------------------------------------------------

## 6.1 Constraint Propagation

For subset constraints:

Let:

    known_criminals
    unknown_tiles

Rules:

    if known_criminals == k_max
        remaining tiles must be innocent

    if known_criminals + unknown_tiles == k_min
        remaining tiles must be criminals

These deductions propagate repeatedly until no changes occur.

------------------------------------------------------------------------

## 6.2 Backtracking Search

Used to verify solutions.

Procedure:

1.  Choose an unknown tile
2.  Try criminal assignment
3.  Try innocent assignment
4.  Propagate constraints
5.  Backtrack on contradictions

The solver stops once:

    2 solutions found

Meaning the puzzle is **not unique**.

------------------------------------------------------------------------

# 7. No‑Guess Verification (Optional)

To ensure puzzles are solvable purely by deduction.

Algorithm:

1.  Start with empty board
2.  Attempt propagation
3.  For each unresolved tile:

Test both assumptions:

    tile = innocent
    tile = criminal

If one assumption leads to contradiction:

The tile value is **forced**.

If no forced tiles exist before puzzle completion:

The puzzle requires guessing → reject.

------------------------------------------------------------------------

# 8. Procedural Generation Pipeline

Puzzle generation follows:

    1. Generate hidden solution
    2. Assign occupations
    3. Create candidate clue pool
    4. Add clues until puzzle becomes unique
    5. Verify logical solvability
    6. Remove redundant clues

------------------------------------------------------------------------

## 8.1 Solution Generation

Random binary grid with target criminal density.

Typical density:

    40–50%

------------------------------------------------------------------------

## 8.2 Occupation Assignment

Sample an occupation histogram first.

Example pattern:

    3,3,3,3,2,2,2,2

Then distribute tiles randomly while avoiding strong clustering.

------------------------------------------------------------------------

## 8.3 Candidate Clue Pool

Generated from the solution.

Typical counts:

  Type                   Count
  ---------------------- -------
  Neighborhood clues     4--8
  Row/column totals      3--5
  Region constraints     2--4
  Occupation relations   1--3
  Row comparisons        1--2

------------------------------------------------------------------------

## 8.4 Add‑Until‑Unique Strategy

Start with a small clue set.

Repeat:

    Add next clue
    Check uniqueness

Stop once:

    solution_count == 1

------------------------------------------------------------------------

## 8.5 Redundancy Pruning

After uniqueness achieved:

Remove clues if:

    puzzle remains unique

This produces **minimal elegant puzzles**.

------------------------------------------------------------------------

# 9. Puzzle Specification Format

Puzzle instances are stored as JSON.

Example:

``` json
{
  "spec_version": 1,
  "grid": {"w":5,"h":4},
  "occupations": {
    "K": 8,
    "occ": [0,1,2,3,4,5,6,7,...]
  },
  "constraints": [
    {"type":"subset","S":[1,2,3],"kmin":1,"kmax":1},
    {"type":"cmp","A":[0,5],"op":">","B":[3,7],"offset":0}
  ]
}
```

The solver uses this representation directly.

------------------------------------------------------------------------

# 10. Performance Targets

For a pure JavaScript implementation:

Typical generation time:

    50ms – 2s

depending on:

-   strict no‑guess checking
-   clue density
-   solver branching

Uniqueness checks remain fast because the board contains only **20
variables**.

------------------------------------------------------------------------

# 11. Future Extensions

Possible improvements:

-   Difficulty scoring based on solve path
-   Controlled clue diversity
-   Region shape libraries
-   Hint generation
-   Puzzle packs

------------------------------------------------------------------------

# 12. Design Philosophy

Key design principles:

1.  **Small rule set**
2.  **Expressive constraints**
3.  **Logical solvability**
4.  **Procedural scalability**
5.  **Single‑file offline execution**

This architecture allows a lightweight generator capable of producing
**thousands of distinct logic puzzles** without external dependencies.

------------------------------------------------------------------------

# 13. Gameplay experience
- Each tile is represented by a card. If the tile has not been marked innocent or guilty, it is grey. If it has been correctly marked as criminal, the background is red. If it is correctly marked as innocent, the background is green.
- When clicked, a modal pops up and asks if that character is innocent or criminal. The player selects one or the other.
- If the player is correct based on the clues shown, the tile's background changes color and the next clue is revealed.
- If the player is not correct or cannot determine the answer based on the clues shown, the board state does not change.
- The entire board should be revealed after no more than 15 clues.
- Once the entire board is revealed, a screen should pop up, showing a grid indicating which tiles the player selected correctly on the first try, vs those that took multiple guesses. It should also contain a button asking the player if they would like to play another round.

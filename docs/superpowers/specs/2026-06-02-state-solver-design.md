# Design: State-based solver (layer-by-layer)

Date: 2026-06-02

## Problem

The current "Vyriešiť" (Solve) button only replays the stored scramble
sequence in reverse (`solveSequence` inverts `scrambleSeq`). This has two
defects:

1. Manual turns done by clicking the cube (`enqueueMoves([move], 'manual')`)
   are never recorded, so the reverse-replay ignores them. After manual
   turns the "solution" leaves the cube unsolved.
2. The button is disabled whenever `scrambleSeq` is empty, so a cube that
   was scrambled only by hand cannot be solved at all.

## Goal

Solve the cube from its **actual current state**, regardless of how it was
scrambled (button scramble, manual clicks, or any mix). The solution must
be correct for any reachable state.

Solution length is not optimized; a layer-by-layer (beginner) method is
acceptable. The animation handles the larger move count.

## Architecture

The solver is **pure logic with no three.js dependency**. The 3D scene
remains the single source of truth for cube state; the solver only reads
from it.

Data flow on "Vyriešiť":

```
3D cubies  ->  readState()      ->  54-char facelet string (URFDLB order)
           ->  solve(facelets)  ->  array of move tokens ['R', "U'", 'F2', ...]
           ->  enqueue(...)      ->  existing animation mechanism
```

This removes the `scrambleSeq` reverse-replay and its disabled-button
condition. "Vyriešiť" is enabled whenever the cube is not solved and no
animation is running.

## Components

Everything stays in a single `index.html`. The solver is an inline block
of pure functions inside the existing `<script type="module">`.

### Solver (inline, pure)

- Logical cube model: a 54-element facelet array with move application for
  `U D L R F B` plus prime and double variants.
- `solve(facelets)` returns an array of move tokens. Beginner method steps:
  1. white cross
  2. white corners (first layer)
  3. second-layer edges
  4. yellow cross (last-layer edge orientation)
  5. last-layer corner orientation
  6. last-layer corner + edge permutation
- Optional move simplification at the end (cancel `U U'`, merge `U U` ->
  `U2`, etc.) to shorten the sequence.
- Internal safety check: the solver simulates its own solution on the
  logical model and verifies the model is solved before returning the
  moves.

The solver's move notation matches the app's existing notation
(`MOVES` + `parseToken`), so the output feeds directly into `enqueue`.

### index.html changes

- `readState()`: build the facelet string from `cubies` (positions,
  `userData.stickers`, `quaternion`). Color -> face letter via `COLORS`.
- `btnSolve` handler: call `solve(readState())` instead of the reverse
  replay.
- `updateButtons()`: enable "Vyriešiť" when `!isBusy && !isSolved()`.
- Remove `scrambleSeq` and `solveSequence()` (orphaned after the change),
  including the `'scramble'` phase's dependency on storing the sequence.

## Error handling / edge cases

- `readState()` assumes a legal (solvable) state. The cube only changes
  through the app's own moves, so the state is always solvable. No handling
  of illegal states (KISS).
- If the cube is already solved, `solve` returns an empty array -> status
  "Kocka je už vyriešená".
- Buttons stay blocked while `isBusy`, as today.

## Testing

The solver is developed and verified outside the browser first: a throwaway
copy of the pure solver logic is run under Node over thousands of random
scrambles, checking that every result ends solved. After it passes, the
verified code lives inline in `index.html`. The Node test file is a
development artifact and is not committed.

## Success criteria

Any random scramble (including manual clicks) -> "Vyriešiť" -> `isSolved()`
returns `true`. Node test: 5000 random scrambles, 100% solved.

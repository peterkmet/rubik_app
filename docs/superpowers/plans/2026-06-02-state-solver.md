# State-based Layer-by-Layer Solver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the scramble-reverse "solver" with a real solver that reads the cube's actual state (including manual click turns) and always solves it.

**Architecture:** A pure-logic cube model (54-facelet array) whose move tables and facelet slots are generated from geometry — identical rotation semantics to the three.js app — so move correctness is by construction. A layer-by-layer (beginner, 2-look last layer) `solve(facelets)` returns move tokens. In the browser, `readState()` maps the 3D cubies to the same facelet layout; tokens feed the existing animation queue. Developed/verified standalone under Node, then inlined into the single `index.html`.

**Tech Stack:** Vanilla ES modules, three.js (browser only, unchanged), Node.js (dev-time test harness only — not shipped).

---

## File structure

- `index.html` — the whole app. Final home of the solver (inline in the existing `<script type="module">`).
- `solver.dev.mjs` — **dev-only** pure solver (model + `solve`). Run under Node. Deleted after inlining.
- `solver.test.mjs` — **dev-only** Node test harness (random-scramble verification). Deleted after inlining.

The two `.mjs` files are development artifacts. They are never committed and are removed in the final task, keeping the deliverable a single `index.html`.

## Notation & semantics (must match the app)

The app defines (index.html:165-172):
```
U: axis y, layer  1, dir -1      R: axis x, layer  1, dir -1
D: axis y, layer -1, dir  1      L: axis x, layer -1, dir  1
F: axis z, layer  1, dir -1      B: axis z, layer -1, dir  1
```
A token's angle is `dir * sign * (π/2) * turns` (index.html:265-277). The solver MUST use these same axis/layer/dir values and the same right-handed rotation as three.js, so that a token emitted by `solve` produces, when animated, exactly the model's predicted state.

Colors -> face letters (index.html:139-146): U=white `0xf7f7f5`, D=yellow `0xffd62e`, F=green `0x4cc23f`, B=blue `0x2f6fed`, R=red `0xe8423a`, L=orange `0xff8b1f`.

---

## Task 1: Cube model — geometry-generated facelets and moves

**Files:**
- Create: `solver.dev.mjs`
- Create: `solver.test.mjs`

- [ ] **Step 1: Create the model in `solver.dev.mjs`**

```js
// Pure cube model. No three.js. Facelet slots and move permutations are
// generated from geometry so they match the 3D app's rotation semantics.
const HALF_PI = Math.PI / 2;

// Move definitions identical to index.html.
export const MOVES = {
  U: { axis: 'y', layer:  1, dir: -1 },
  D: { axis: 'y', layer: -1, dir:  1 },
  R: { axis: 'x', layer:  1, dir: -1 },
  L: { axis: 'x', layer: -1, dir:  1 },
  F: { axis: 'z', layer:  1, dir: -1 },
  B: { axis: 'z', layer: -1, dir:  1 },
};

const AXES = ['x', 'y', 'z'];
const v = (x, y, z) => ({ x, y, z });
const keyOf = (p, n) => `${p.x},${p.y},${p.z}|${n.x},${n.y},${n.z}`;

// Right-handed rotation by k*90deg about an axis (integer, drift-free).
function rotate(p, axis, k) {
  let { x, y, z } = p;
  for (let i = 0; i < ((k % 4) + 4) % 4; i++) {
    if (axis === 'x') { const ny = -z, nz = y; y = ny; z = nz; }
    else if (axis === 'y') { const nx = z, nz = -x; x = nx; z = nz; }
    else { const nx = -y, ny = x; x = nx; y = ny; }
  }
  return v(x, y, z);
}

// 6 faces in URFDLB order, each with its outward normal axis/sign.
const FACE_DEFS = [
  { face: 'U', nrm: v(0, 1, 0) },
  { face: 'R', nrm: v(1, 0, 0) },
  { face: 'F', nrm: v(0, 0, 1) },
  { face: 'D', nrm: v(0,-1, 0) },
  { face: 'L', nrm: v(-1,0, 0) },
  { face: 'B', nrm: v(0, 0,-1) },
];

// Build the 54 facelet slots: for each face, the 9 surface stickers.
// Order within a face is deterministic (sorted by the two in-plane axes).
export const FACELETS = [];
for (const { face, nrm } of FACE_DEFS) {
  const fixedAxis = AXES.find((a) => Math.abs(nrm[a]) === 1);
  const plane = AXES.filter((a) => a !== fixedAxis);
  const slots = [];
  for (const a0 of [-1, 0, 1])
    for (const a1 of [-1, 0, 1]) {
      const p = v(0, 0, 0);
      p[fixedAxis] = nrm[fixedAxis];
      p[plane[0]] = a0;
      p[plane[1]] = a1;
      slots.push({ pos: p, nrm, face });
    }
  FACELETS.push(...slots);
}

// index lookup by (pos,nrm)
const SLOT_INDEX = new Map();
FACELETS.forEach((s, i) => SLOT_INDEX.set(keyOf(s.pos, s.nrm), i));

// Precompute a permutation array for each quarter-turn face move.
// perm[j] = i means: after the move, slot j holds the sticker that was in slot i.
function buildPerm(face) {
  const { axis, layer, dir } = MOVES[face];
  const perm = FACELETS.map((_, i) => i); // identity for untouched slots
  FACELETS.forEach((s, i) => {
    if (s.pos[axis] !== layer) return;
    // dir = +1 -> k=+1 quarter turn about axis (matches three.js angle sign)
    const np = rotate(s.pos, axis, dir);
    const nn = rotate(s.nrm, axis, dir);
    const j = SLOT_INDEX.get(keyOf(np, nn));
    perm[j] = i;
  });
  return perm;
}
const PERM = {};
for (const f of Object.keys(MOVES)) PERM[f] = buildPerm(f);

export const SOLVED = FACELETS.map((s) => s.face);

// Apply a single quarter turn (face letter) to a facelet array, returns new array.
function turn(state, face) {
  const perm = PERM[face];
  return perm.map((i) => state[i]);
}

// token -> apply to state (handles '', "'", '2')
export function applyToken(state, token) {
  const face = token[0];
  const suffix = token.slice(1);
  let s = state;
  const times = suffix === '2' ? 2 : 1;
  // prime = inverse = 3 quarter turns
  const reps = suffix === "'" ? 3 : times;
  for (let i = 0; i < reps; i++) s = turn(s, face);
  return s;
}

export function applyTokens(state, tokens) {
  return tokens.reduce(applyToken, state);
}

export function isSolved(state) {
  return state.every((c, i) => c === SOLVED[i]);
}
```

- [ ] **Step 2: Write the failing model tests in `solver.test.mjs`**

```js
import assert from 'node:assert';
import { SOLVED, FACELETS, applyToken, applyTokens, isSolved, MOVES } from './solver.dev.mjs';

let passed = 0;
function test(name, fn) { fn(); passed++; console.log('ok -', name); }

test('54 facelets, 6 colors x 9', () => {
  assert.equal(FACELETS.length, 54);
  for (const f of Object.keys(MOVES)) {
    assert.equal(SOLVED.filter((c) => c === f).length, 9);
  }
});

test('solved state is solved', () => assert.ok(isSolved(SOLVED)));

test('a quarter turn unsolves, four restore', () => {
  for (const f of Object.keys(MOVES)) {
    let s = applyToken(SOLVED, f);
    assert.ok(!isSolved(s), `${f} should unsolve`);
    s = applyTokens(SOLVED, [f, f, f, f]);
    assert.ok(isSolved(s), `${f}x4 should restore`);
  }
});

test('move and its inverse cancel', () => {
  for (const f of Object.keys(MOVES)) {
    assert.ok(isSolved(applyTokens(SOLVED, [f, f + "'"])), `${f} ${f}' cancels`);
    assert.ok(isSolved(applyTokens(SOLVED, [f + '2', f + '2'])), `${f}2 ${f}2 cancels`);
  }
});

test('scramble then inverse-scramble solves', () => {
  const faces = Object.keys(MOVES); const suff = ['', "'", '2'];
  for (let t = 0; t < 200; t++) {
    const seq = [];
    for (let i = 0; i < 25; i++)
      seq.push(faces[(Math.random()*6)|0] + suff[(Math.random()*3)|0]);
    const scrambled = applyTokens(SOLVED, seq);
    const inv = [...seq].reverse().map((tok) => {
      const f = tok[0], s = tok.slice(1);
      return s === '2' ? tok : s === "'" ? f : f + "'";
    });
    assert.ok(isSolved(applyTokens(scrambled, inv)), 'inverse should solve');
  }
});

console.log(`\n${passed} model tests passed`);
```

- [ ] **Step 3: Run tests, expect failure first (file not yet importable / logic gaps)**

Run: `node solver.test.mjs`
Expected at first run: PASS only if Step 1 is correct. If any test fails (e.g. a rotation sign), fix `rotate`/`buildPerm` until all pass. The four-turn and inverse tests catch sign errors.

- [ ] **Step 4: Run tests, expect pass**

Run: `node solver.test.mjs`
Expected: all model tests pass.

- [ ] **Step 5: Commit (dev artifacts excluded — commit nothing yet)**

No commit. `solver.dev.mjs` and `solver.test.mjs` are uncommitted dev artifacts. Verification is the green test run.

---

## Task 2: Layer-by-layer `solve(facelets)`

Implement `solve` incrementally, one phase per step, each gated by a random-state Node test asserting that phase's invariant. The solver works on a mutable copy, records emitted tokens, and applies each emitted token to the working state so detection always reflects the current state.

**Files:**
- Modify: `solver.dev.mjs` (add `solve` and helpers)
- Modify: `solver.test.mjs` (add per-phase + end-to-end tests)

Shared helpers to add to `solver.dev.mjs`:

```js
// Solver works with a context that mutates state and logs moves.
function makeCtx(state) {
  const ctx = { state: state.slice(), moves: [] };
  ctx.do = (seq) => {
    const tokens = Array.isArray(seq) ? seq : seq.trim().split(/\s+/).filter(Boolean);
    for (const t of tokens) { ctx.state = applyToken(ctx.state, t); ctx.moves.push(t); }
  };
  return ctx;
}

// color at a slot identified by (pos,nrm)
function colorAt(state, pos, nrm) {
  return state[SLOT_INDEX.get(keyOf(pos, nrm))];
}
```

Phases (last layer = U, first layer built on D; standard beginner method):

- [ ] **Step 1: Phase 1 — D cross.** Place the four D-layer edges (D + side color) so the D face shows a cross and each edge's side sticker matches its side center. Implement by, for each of the 4 D edges, locating it, bringing it to the U layer, aligning above its target, and inserting with `F2`/`R2`/etc. (or the appropriate 3-move insert when flipped). Loop until all four are placed.

  Test (add to `solver.test.mjs`): over 2000 random 25-move scrambles, run Phase 1 only and assert the four D-edge slots and their side stickers are correct. Run `node solver.test.mjs`; iterate code until green.

- [ ] **Step 2: Phase 2 — D corners.** For each of the 4 D corners: bring it under its target slot in the U layer, then apply the `R U R'` / `R U' R'` repeat (sexy-move) insertion until seated correctly. Loop until the whole D layer is solved.

  Test: random scrambles, after Phases 1+2 assert the entire D face and the bottom ring of side stickers are solved. Iterate to green.

- [ ] **Step 3: Phase 3 — middle layer edges.** For each non-U, non-D edge: position it in the U layer above the matching center, then apply the left/right insert algorithm:
  - right insert: `U R U' R' U' F' U F`
  - left insert: `U' L' U L U F U' F'`
  - if the edge is stuck in the middle wrong, insert any U edge to evict it first.
  Loop until both bottom two layers are solved.

  Test: after Phases 1-3 assert bottom two layers solved. Iterate to green.

- [ ] **Step 4: Phase 4 — U cross (orient LL edges).** Using `F R U R' U' F'` from the relevant angle, turn the dot/L/line state into a U cross (all four U-edge stickers = U color). Apply, re-detect, repeat (max 3 applications).

  Test: after Phases 1-4 assert all four U-edge top stickers are U. Iterate to green.

- [ ] **Step 5: Phase 5 — orient LL corners.** Using the repeated `R U R' U R U2 R'` (Sune) from the correctly chosen corner, make all four U-corner top stickers = U. Apply/rotate U/repeat until the U face is solid.

  Test: after Phases 1-5 assert the entire U face is U color. Iterate to green.

- [ ] **Step 6: Phase 6 — permute LL corners then edges.** 
  - Corners: rotate U to find a solved (or any) corner, apply corner-cycle `U R U' L' U R' U' L` until all four corners are in place.
  - Edges: apply edge-cycle `R U' R U R U R U' R' U' R2` (U-perm) with the correct setup `U` rotations until all four edges are placed.
  - Finish with a final `U/U'/U2` if needed to align the top layer.

  Test (end-to-end): over 5000 random 25-move scrambles, `solve(applyTokens(SOLVED, scramble))` then assert `isSolved(applyTokens(scrambled, solution))`. Iterate to green.

- [ ] **Step 7: Add the internal safety check and simplification, then export `solve`**

```js
// Cancel/merge consecutive same-face moves: e.g. U U' -> nothing, U U -> U2,
// U2 U2 -> nothing, U U2 -> U'.
function simplify(tokens) {
  const out = [];
  for (const t of tokens) {
    const prev = out[out.length - 1];
    if (prev && prev[0] === t[0]) {
      const amt = (q) => (q.slice(1) === '2' ? 2 : q.slice(1) === "'" ? 3 : 1);
      const total = (amt(prev) + amt(t)) % 4;
      out.pop();
      if (total === 1) out.push(t[0]);
      else if (total === 2) out.push(t[0] + '2');
      else if (total === 3) out.push(t[0] + "'");
    } else out.push(t);
  }
  return out;
}

export function solve(state) {
  const ctx = makeCtx(state);
  phaseDCross(ctx);
  phaseDCorners(ctx);
  phaseMiddle(ctx);
  phaseUCross(ctx);
  phaseUCorners(ctx);
  phasePermute(ctx);
  const moves = simplify(ctx.moves);
  // safety: simplified solution must solve the original state
  if (!isSolved(applyTokens(state, moves)))
    throw new Error('solver produced an invalid solution');
  return moves;
}
```

  Run: `node solver.test.mjs`
  Expected: all phase tests + the 5000-scramble end-to-end test pass; no "invalid solution" throw.

- [ ] **Step 8: Commit**

No commit (dev artifacts). The green 5000-scramble run is the verification.

---

## Task 3: Inline the solver into `index.html` and wire it up

**Files:**
- Modify: `index.html` (script module: add solver inline + `readState`, change `btnSolve`, `updateButtons`; remove `scrambleSeq`/`solveSequence`)

- [ ] **Step 1: Inline the verified solver**

Paste the verified contents of `solver.dev.mjs` (without the `export` keywords needed only for Node) into the top of the existing `<script type="module">` in `index.html`, after the imports. Keep `MOVES` single-sourced: the app already declares `MOVES` (index.html:165-172) and `HALF_PI` (index.html:152) — reuse those, do not redeclare. Bring in only the solver-specific additions: `rotate`, `FACE_DEFS`-based facelet build, `FACELETS`, `SLOT_INDEX`, `PERM`, `SOLVED`, `turn`, `applyToken`, `applyTokens`, model `isSolved` (rename to avoid clashing with the existing 3D `isSolved` — call it `modelSolved`), `makeCtx`, `colorAt`, the phase functions, `simplify`, and `solve`.

- [ ] **Step 2: Add `readState()` mapping 3D cubies -> facelet array**

```js
// Build the model facelet array from the live 3D scene, using the SAME slot
// definitions (pos,nrm) as the model, so alignment is automatic.
const COLOR_TO_FACE = Object.fromEntries(
  Object.entries(COLORS).map(([f, c]) => [c, f])
);
function readState() {
  const state = new Array(54);
  // index cubies by rounded grid position
  const at = new Map();
  cubies.forEach((c) => {
    const p = `${Math.round(c.position.x)},${Math.round(c.position.y)},${Math.round(c.position.z)}`;
    at.set(p, c);
  });
  FACELETS.forEach((slot, i) => {
    const c = at.get(`${slot.pos.x},${slot.pos.y},${slot.pos.z}`);
    // find the sticker whose world normal matches slot.nrm
    const want = new THREE.Vector3(slot.nrm.x, slot.nrm.y, slot.nrm.z);
    let face = null;
    for (const s of c.userData.stickers) {
      const wn = s.normal.clone().applyQuaternion(c.quaternion);
      if (wn.dot(want) > 0.9) { face = COLOR_TO_FACE[s.color]; break; }
    }
    state[i] = face;
  });
  return state;
}
```

- [ ] **Step 3: Browser self-check — `readState` matches the model after a known scramble**

Add a temporary debug hook on `window.rubik` (remove before final commit):
```js
window.rubik.check = () => {
  const seq = ['R', "U'", 'F2', 'L', 'B', "D'", 'R2', 'U'];
  // apply to model from solved
  let m = SOLVED.slice();
  seq.forEach((t) => { m = applyToken(m, t); });
  // apply same to the 3D cube instantly, then compare
  console.log('expected model vs readState — run after enqueue(seq) finishes');
  return { model: m };
};
```
Verify in browser console: reset cube, run `rubik.applyMove` for each token (or `enqueue(seq,'manual')`), then compare `readState()` to the model array. They must be identical. Fixing any mismatch here means a rotation-sign or slot-order mismatch between model and 3D — adjust `rotate`/sign so the model matches the app (the app's animation is the source of truth).

- [ ] **Step 4: Rewire `btnSolve` to use the real solver**

Replace the handler at index.html:417-426:
```js
btnSolve.addEventListener('click', () => {
  if (isBusy || isSolved()) return;
  const solution = solve(readState());
  if (solution.length === 0) { setStatus('Kocka je už <b>vyriešená</b> ✓'); return; }
  setStatus(`Riešim… <b>${solution.length}</b> ťahov`);
  enqueue(solution, 'solve', () => {
    setStatus(isSolved() ? 'Kocka je <b>vyriešená</b> ✓' : 'Hotovo.');
    updateButtons();
  });
});
```

- [ ] **Step 5: Update `updateButtons` and remove dead scramble-replay code**

Change `updateButtons` (index.html:398-402):
```js
function updateButtons() {
  btnScramble.disabled = isBusy;
  btnReset.disabled = isBusy;
  btnSolve.disabled = isBusy || isSolved();
}
```
Remove now-orphaned code:
- `let scrambleSeq = [];` (index.html:345) and all assignments/uses of `scrambleSeq`.
- `function solveSequence(...)` (index.html:362-364).
- In `btnScramble` handler, replace `scrambleSeq = generateScramble(n); ... enqueue(scrambleSeq, ...)` with a local `const seq = generateScramble(n);` and `enqueue(seq, 'scramble', ...)`. Keep `generateScramble`.
- In `btnReset` and `btnSolve` done-callbacks remove `scrambleSeq = []`.
Because `solve` reads live state, scrambling no longer needs to store anything.

Call `updateButtons()` after each move completes so "Vyriešiť" enables right after a manual turn. In the render loop's `isBusy -> false` branch (index.html:449-453) `updateButtons()` is already called — confirm it runs after `'manual'` moves too (it does, since manual moves set `isBusy`).

- [ ] **Step 6: Manual browser verification**

Run a local server and open the app:
```bash
python3 -m http.server 8000
```
- Scramble with the button -> "Vyriešiť" -> cube ends solved, status shows ✓.
- Reset, then manually click ~15 turns -> "Vyriešiť" stays enabled -> solves to ✓.
- Mix: button-scramble + manual clicks -> "Vyriešiť" -> ✓.
- Already solved -> "Vyriešiť" disabled.

- [ ] **Step 7: Remove the temporary `window.rubik.check` debug hook**

Delete the hook added in Step 3.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: state-based layer-by-layer solver (solves any state, incl. manual turns)"
```

---

## Task 4: Update documentation and clean up dev artifacts

**Files:**
- Modify: `docs/PRG.md` (section 4.2 — solver description)
- Modify: `CLAUDE.md` (rubik_app — solver/architecture note)
- Delete: `solver.dev.mjs`, `solver.test.mjs`

- [ ] **Step 1: Update `docs/PRG.md` section 4.2**

Replace the "spätné prehranie" description (PRG.md:96-108) with: the solver reads the cube's current state via `readState()` into a 54-facelet model and solves it with a layer-by-layer (beginner, 2-look last layer) algorithm; works from any state, including manual click turns; solution is simplified (cancel/merge consecutive same-face turns). Note "Vyriešiť" is enabled whenever the cube is not solved. Keep Slovak (PRG.md is Slovak).

- [ ] **Step 2: Update `rubik_app/CLAUDE.md`**

Update the "Solver" bullet in the architecture summary: solver reads live state into a facelet model and solves layer-by-layer (no longer scramble rewind). Remove the "Plánované" mention of solver-as-rewind if present. Keep Slovak.

- [ ] **Step 3: Delete dev artifacts**

```bash
rm solver.dev.mjs solver.test.mjs
```

- [ ] **Step 4: Verify single-file state and no stray references**

Run: `git status` and `grep -n "solver.dev\|solver.test\|scrambleSeq\|solveSequence" index.html`
Expected: no matches in `index.html`; `.mjs` files gone.

- [ ] **Step 5: Commit**

```bash
git add docs/PRG.md CLAUDE.md
git commit -m "docs: describe state-based solver"
```

---

## Self-review notes

- **Spec coverage:** pure no-three.js solver (Task 1-2), facelet URFDLB-style layout from geometry (Task 1), `readState` mapping (Task 3 Step 2), `solve` returns tokens (Task 2), button enabled when not solved (Task 3 Step 5), already-solved -> empty/✓ (Task 3 Step 4), internal safety check (Task 2 Step 7), Node 5000-scramble test (Task 2 Step 6), single `index.html` deliverable with dev artifacts removed (Task 4). All covered.
- **Naming:** model solved-check is `modelSolved`/`isSolved` in the dev module but renamed to avoid clash with the 3D `isSolved` when inlined (Task 3 Step 1) — the 3D `isSolved` stays the success oracle in the browser; the model uses its own. `generateScramble` kept; `scrambleSeq`/`solveSequence` removed.
- **Notation:** solver `MOVES`/`dir` mirror index.html exactly so emitted tokens animate to the predicted state; rotation sign verified by Task 1 four-turn/inverse tests and Task 3 Step 3 browser comparison.

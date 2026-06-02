# vendor/

Third-party code, vendored locally (no CDN at runtime).

## cubejs — `cube.js`, `solve.js`

Rubik's Cube model and Kociemba two-phase solver.

- Source: https://github.com/ldez/cubejs (npm package `cubejs`, v1.3.0)
- License: MIT
- Used by `index.html` as global `<script>` includes. `Cube.initSolver()`
  builds the pruning tables (lazy, on first solve); `Cube.fromString(facelets).solve()`
  returns a ~20-move solution.

Loaded as plain globals before the app's ES module, so `window.Cube` is
available to the module.

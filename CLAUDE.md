# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Tetris implemented in vanilla JavaScript with HTML5 Canvas. No dependencies, no build step, no package.json. Just three files: `index.html`, `style.css`, `game.js`.

## Running / testing

There is no build, lint, or test tooling in this repo. To run the game, serve the directory and open it in a browser:

```bash
python3 -m http.server 8000   # or: npx serve .
```

Then open `http://localhost:8000`. Opening `index.html` directly via `file://` also works since there are no modules or fetch calls.

There are no automated tests. Verify changes manually in the browser (see the `/verify` skill workflow): check movement, rotation/wall-kicks, soft/hard drop, line clearing, level speed-up, pause, and game-over/restart.

## Architecture (`game.js`)

Everything lives in one file with module-level mutable state (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.) — there's no class/component structure to preserve, so keep new logic consistent with this flat, procedural style rather than introducing abstractions.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–7` identifying which piece locked there.
- **Pieces**: `PIECES` are hardcoded square matrices (index 0 unused). `current`/`next` pieces are `{ type, shape, x, y }`. Rotation (`rotateCW`) is a generic matrix transpose+reverse — it works on any square shape, so new piece shapes don't need custom rotation logic.
- **Collision** (`collide`): checked against board bounds and locked cells; used everywhere before mutating a piece's position (movement, rotation, gravity, ghost projection).
- **Wall kicks** (`tryRotate`): after rotating, tries x-offsets `[0, -1, 1, -2, 2]` in order and takes the first that doesn't collide.
- **Lock → clear → spawn pipeline** (`lockPiece`): `merge()` bakes the current piece into `board`, `clearLines()` removes full rows (bottom-up, re-checks the same row index after a splice since rows shift down), then `spawn()` promotes `next` to `current` and generates a new `next`. If the freshly spawned piece immediately collides, `endGame()` fires.
- **Scoring/leveling**: `LINE_SCORES = [0, 100, 300, 500, 800]` indexed by lines-cleared-at-once, multiplied by `level`. `level` increases every 10 total lines; `dropInterval = max(100, 1000 - (level-1)*90)` ms controls gravity speed.
- **Game loop** (`loop`, driven by `requestAnimationFrame`): accumulates elapsed time in `dropAccum`; once it exceeds `dropInterval`, advances the piece one row or locks it. `draw()` runs every frame regardless (grid → locked board → ghost piece at 20% alpha → current piece).
- **Input** is a single `keydown` listener that no-ops when `paused || gameOver` (except `KeyP`, which always toggles pause).

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).

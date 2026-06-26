# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

A **single-file HTML5 Canvas 2D isometric idle-arcade ad mini-game** themed on
*Whiteout Survival* (《寒霜啟示錄》). It is a playable advertising creative /
standalone web mini-game, **not** the SLG it advertises.

Core loop: **採集 (gather) → 搬運 (carry) → 交付解鎖 (deposit-to-unlock) →
加工 (process) → 販售 (sell) → 升級 (upgrade) → 擴張 (expand)**.

The entire game is `index.html` — there is **no build step, no dependencies,
no framework, no package.json, no tests**. Plain HTML + CSS + vanilla JS in one
file (~480 lines).

## Repository layout

| File | Purpose |
|---|---|
| `index.html` | **The entire game.** Single source of truth for all behaviour & numbers. |
| `GAMEPLAY_SPEC.md` | Gameplay design spec (Traditional Chinese). **§0 = authoritative as-built**; §1+ is forward-looking type research (proposed values, not necessarily current). |
| `經濟與單位定義.md` | Economy & unit definitions (Traditional Chinese), aligned to spec §0. |
| `.gitignore` | Ignores `.DS_Store`, `.claude/`, `node_modules/`. |

The two `.md` files are **design docs that describe `index.html`**. When the
code and the docs disagree, **`index.html` is the truth** — then update the docs
to match (see "When you change the game" below).

## Running / testing

- **Run:** open `index.html` directly in a browser (e.g. `file://` path or any
  static server). No install, no build.
- **No automated tests / linters exist.** "Testing" = loading the page and
  playing. The fixed-timestep loop and isometric rendering mean visual/manual
  verification is the practical check. Playwright + Chromium are available in
  this environment (`/opt/pw-browsers/chromium`) if you need to drive the page
  programmatically or screenshot.
- Designed for **portrait** (~9:16, `width:min(100vw, 100vh*0.5625)`), works on
  desktop + touch.

## Code architecture (`index.html`)

All JS lives in one IIFE at the bottom of the file. Key landmarks (line numbers
drift — search by name):

- **Global state `S`** (~line 59): the whole game lives here — `money`, `player`,
  `carryCap`, `gatherInt`, `moveSpeed`, `procInt`, `spawnInt`, `serveInt`,
  `workers[]`, `cows[]`, `stations[]`, `floats[]`, `cubes[]`, etc.
- **Isometric projection** (`iso`, `oX`, `oY`, `TILE_W=44`, `TILE_H=30`,
  `GRID=14`): `x=oX+(c−r)·22, y=oY+(c+r)·15`; depth-sorted by `c+r`.
- **`recomputeStats()`** (~line 101): **the balance hub.** Derives *every*
  level-scaled value (capacities, speeds, costs, worker counts, prices, queue
  caps) from upgrade levels via formulas. **To tune balance, edit the formulas
  here** — not scattered constants.
- **`spreadStations()`** (~line 137): auto-layout. Stations are placed by AABB
  minimal-translation separation so labels never overlap or leave the screen.
  Runs once on first draw. You don't hand-place coordinates.
- **`objective()`** (~line 164): per-frame "what to do now" state machine →
  top banner text + target highlight/arrow.
- **Systems:** `harvestCows`, `updateProc` (processors), `updateCounter` (sell
  counter / FIFO customers), `updateCows`, `updateWorker` (worker AI), `update`
  (main tick).
- **Worker AI:** three auto roles — `lumber` (伐木工), `rancher` (牧場工),
  `clerk` (運輸工/店員, split into meat `clerkB` & ration `clerkR`). See
  `woodDests()`, `pickDest()`, `doUnload()`. Designed to avoid clumping/jitter.
- **Rendering:** `draw`, `drawStation`, `drawTile`, `person`, `drawCow`,
  `drawPile`, `drawBar`; HUD via `refreshHUD`, `toast`.
- **Input:** virtual joystick (`jStart`/`jMove`/`jEnd`, drag to move player).
- **Main loop** (~line 478): `requestAnimationFrame` + accumulator, fixed
  `STEP=1/60`, single-frame `dt` clamped to `0.1` (background-tab throttle
  compensation).
- **Save/load/reset** (~line 442): `localStorage` key `whiteout_save_v1`,
  payload **version `v:3`**. Auto-saves every ~2s and on `visibilitychange`.

## Key domain conventions

- **Resources:** `wood` (raw), `cattle` (raw), `beef` (product), `ration`
  (product). `PRODUCTS=['beef','ration']`, `PRICE={beef:12, ration:26}`.
- **Levels are 1-indexed:** `Lv.1 = unupgraded (base)`; formulas use `(lvl−1)`.
  7 upgrade stations: `up_player`, `up_lumber`, `up_rancher`, `up_clerk`,
  `up_clerk2`, `up_proc`, `up_spawn`. Cost `= round(base × growth^(lvl−1))`.
- **Deposit-to-unlock replaces a warehouse:** build frames `frame1..frame4`
  unlock in sequence; deliver the *right* material (wrong type rejected) to fill
  and unlock the next content.
- **Per-material carry caps:** `carryCap` (start 12) applies *per material*
  independently. Workers use a soft cap `wcap()`.
- **Balance philosophy (from the docs):** tune by lowering `base` cost only;
  **don't touch `growth`** (in a 1–3 min ad session, `base` controls
  first-worker timing = high leverage; `growth` controls late-game inflation =
  high risk).
- **Save only stores levels + raw progress** (money, cow count, locked flags,
  frame progress, upgrade `lvl`+`acc`, buffers, counter stock). Level-derived
  values are **not** saved — `recomputeStats()` recomputes them on load. This is
  deliberate: change a formula and old saves auto-rebalance.

## When you change the game

1. Make the change in `index.html` (this is the only code file).
2. If balance/mechanics changed, **update the formulas in `recomputeStats()`**
   rather than hardcoding values elsewhere.
3. Verify by loading the page in a browser and playing through the relevant flow
   (frame unlock chain, selling, upgrades). There's no test suite to run.
4. **Keep the docs in sync:** update `GAMEPLAY_SPEC.md` §0 (as-built) and
   `經濟與單位定義.md` to reflect new numbers/behaviour. These files explicitly
   claim to mirror the shipped `index.html`. Note: the docs currently describe
   the save version as `v1/v2` in prose while the code uses `v:3` — trust the
   code and prefer fixing such drift when you touch related areas.
5. Comments and docs are in **Traditional Chinese**; match that style. UI strings
   are Traditional Chinese.

## Git / workflow

- Active development branch: **`claude/claude-md-docs-tyrzoa`** (already checked
  out). Develop, commit, and push here. Push with
  `git push -u origin claude/claude-md-docs-tyrzoa`.
- Do **not** push to `main` without explicit permission.
- Do **not** open a pull request unless explicitly asked.
- Commit messages: clear and descriptive (Chinese or English both fine, matching
  the existing `寒霜啟示錄...` style is welcome).

## Gotchas

- `Date.now()` / `Math.random()` are fine in this game code (browser runtime) —
  the restriction in this harness applies to workflow scripts, not `index.html`.
- The CTA "立即遊玩" button and ✕ close are static HTML overlay elements; there
  is **no `adMode` parameter implemented in code** despite the spec mentioning
  one as a future toggle.
- Station positions are computed, not authored — don't expect fixed coordinates.
- Large-number formatting goes through `fmt()`.
</content>
</invoke>

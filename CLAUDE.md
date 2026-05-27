# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Two standalone browser games, each a single self-contained HTML file (no build step, no dependencies, no server required). Open any file directly in a browser to run it.

- **`game.html`** — Arrow Siege: retro 2D bow-and-arrow shooter
- **`tictactoe.html`** — Two-player Tic Tac Toe with score tracking

## Running the games

```
start game.html        # Windows — opens in default browser
start tictactoe.html
```

No build, no install, no localhost needed.

## Git workflow

After every meaningful change:
1. Stage the specific file(s) changed — never `git add .` blindly
2. Write a commit message: imperative subject line, body describes *why* not just *what*
3. Push to `origin/main`

```
git add game.html
git commit -m "short imperative summary

Optional longer body explaining the why."
git push
```

Remote: https://github.com/Vijaythri/arrow-siege

## game.html architecture

Everything lives in one `<script>` block, divided by comment headers:

| Section | Responsibility |
|---------|---------------|
| 1. Constants | `CANVAS_W/H`, physics values, `E` (enemy type enum), `ECFG` (per-type stats), `S` (state enum) |
| 2. Canvas setup | Fixed 640×480 internal size; CSS scaling via `resize()` on window resize |
| 3. Input | `inp` object polled each frame — `keys{}`, `mx/my`, `mdown`, `mclick`. `mclick` is cleared at the **end** of every frame |
| 4. Web Audio | `ensureAudio()` lazily creates `AudioContext` on first user gesture. `snd(type)` synthesizes all sounds with Oscillator+GainNode; 50 ms per-type throttle prevents glitch on rapid collisions |
| 5. Level configs | `getLvlCfg(n)` returns 5 hand-crafted levels then procedurally scales. `buildQueue(cfg)` flattens wave arrays into a time-sorted spawn list |
| 6. Game state | `state`, `stateT`, `player`, `enemies[]`, `pArrows[]`, `eArrows[]`, `parts[]`, `lvlSt` are the live game objects |
| 7. Update | `updateGame(dt)` calls sub-updaters in order: player → spawner → enemies → arrows → collisions → particles → filter dead entities → check win/lose |
| 8. Draw | `renderGame()` renders back-to-front: background → particles → enemy arrows → enemies → player arrows → player → HUD |
| 9. Screens | `renderMenu()`, `renderLevelComplete()`, `renderGameOver()` — all drawn on canvas with no DOM elements |
| 10–11. Management | `startLevel(n)` / `resetMenu()` reset live arrays and transition state. Main `loop(ts)` dispatches on `state` |

### Key patterns

**Entity shape — enemy:**
```js
{ x, y, hp, maxHp, size, type, speed,
  aFrame, aTimer,          // animation
  dTimer, dead, dParts,    // death sequence
  flash,                   // hit-flash timer
  shootT, rumbleT, reward }
```

**Entity removal** — always filter *after* the loop, never splice during:
```js
enemies = enemies.filter(e => !e.dead || e.dTimer > 0);
```

**Mouse coordinates** must be corrected for CSS scaling:
```js
inp.mx = (e.clientX - r.left) * (CANVAS_W / r.width);
```

**Bow angle** — `Math.atan2(inp.my - player.y, inp.mx - player.x)` — recomputed every frame; arrow inherits this angle on spawn.

**Arrow gravity** — applied each frame to `vy` only (`vy += GRAVITY * dt`); `angle` is then recomputed from `atan2(vy, vx)` so the shaft visually rotates downward.

**`inp.mclick`** is a one-shot flag. It is set on `mousedown` and cleared at the very end of `loop()`. All state transitions that need to consume a click must read it before the loop clears it.

### Adding a new enemy type

1. Add a key to `E` and an entry to `ECFG` (speed, hp, reward, size, fps)
2. Write a `drawXxx(e, bx, by)` function
3. Add a case to `drawEnemy()` and the `deathParts` color map
4. Add AI logic in `updateEnemies()` under a new `else if (e.type === E.XXX)` branch
5. Reference the type in one or more level wave configs in `getLvlCfg()`

### Tuning gameplay

All physics and timing constants are at the top of the file (Section 1). Level difficulty (enemy counts, spawn gaps, speed scaling per level) lives entirely in `getLvlCfg()`.

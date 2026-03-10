# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

Open `index.html` directly in a browser. There is no build system, no dependencies, and no package manager. Everything is in the single file.

## Architecture

The entire game is a single self-contained file: `index.html`. It contains CSS, HTML markup, and JavaScript in one `<script>` block.

### Data Definitions (top of script)

All game content is defined as plain arrays/objects at the top of the script:

- **`DEFS[]`** — 10 attack definitions (id 0–9: Fireball, Ice Spike, Lightning, Whirlwind, Shockwave, Arrow Volley, Earthquake, Poison Nova, Laser Beam, Meteor). Each has `stat(level)` returning computed stats and `fire(S, level)` for attack logic. Laser Beam (id 8) is special — it has no `fire()` and is handled directly in `update()` and `render()`.
- **`PERKS[]`** — 6 stackable permanent upgrades chosen at milestones/mastery. Each has `apply(p)` which mutates the player object.
- **`PASSIVES[]`** — 3 stackable passive abilities (lifesteal, crit, evasion).
- **`ULTIMATES[]`** — 5 ultimate abilities activated with SPACE. Each has `activate(S)`.
- **`ITEM_DEFS`** — 3 pickup types (heal, dmg, spd).
- **`BG_DECORS[]`** — 700 static background decorations, generated once at startup via a seeded PRNG.

### State Object (`S`)

Created by `mkState()`, reset on restart:

```
S.p          — player: wx/wy (world position), hp, maxHp, spd, level, xp, kills,
               dmgMult/spdMult (active buff multipliers),
               permDmgMult/permSpdMult/dmgReduction/regenRate/cdReduction (perk stats),
               perks{}, passives{}, chosenUlt, ultCharge, ultCd, mdx/mdy (last move dir),
               specialAtk (attack id or null), specialCd (ms remaining)
S.atks[]     — active auto-firing attacks: {id, level, cd}
S.enemies[]  — enemy objects
S.projs[]    — projectiles
S.fxs[]      — visual + gameplay effects (shockwave, poison, meteor, particles, etc.)
S.items[]    — world pickups
S.buffs      — timed buff countdowns: {dmg, spd} in ms
S.pendingPerks/pendingPassives — queued selection screens
```

### Game Loop

`requestAnimationFrame` → `loop(ts)` → `update(dt)` + `render()`. Delta time `dt` is capped at 50ms.

### Coordinate System

World is 3000×3000 (`W = 3000`) centered at (0,0). Player `wx/wy` are world coordinates. `ws(wx, wy)` converts to screen coordinates (player is always centered on screen).

### Effect System

`S.fxs[]` is a unified array handling both visual particles and gameplay effects. Effects have `type`, `life` (ms remaining), and type-specific fields. They are both processed in `update()` (e.g., shockwave deals damage as it expands, poison ticks damage) and rendered in `render()`.

### HUD vs Canvas

All HUD elements (HP/XP bars, buff pills, attack bar, ultimate bar, overlays) are DOM elements positioned with CSS `position:fixed`. The `<canvas>` renders only the game world. UI updates happen via direct DOM manipulation in `updateHUD()`, `renderBar()`, `updateBuffRow()`, `updateUltBar()`, `updatePerkBadges()`.

### Special Attack (E key)

At game start the player picks 1 of 5 random attacks as their **special attack**, stored in `S.p.specialAtk` (attack id). It does not appear in `S.atks[]` and does not auto-fire. Pressing `E` calls `activateSpecial()`, which fires the attack at level 1 and resets `S.p.specialCd = 5000` (ms). The cooldown ticks in `update()`. The `#special-slot` DOM element renders the slot to the left of the attack bar, showing the icon, a countdown, and a cooldown mask.

### Progression Flow

On game start: perk selection → passive selection → ultimate selection → special attack selection (E key) → first auto-attack level-up → game begins.

On level-up: attack upgrade/new attack choice. Every 10 levels also queues a perk + passive selection. Mastering an attack (reaching level 10) queues a perk selection. `pendingPerks`/`pendingPassives` track the queue; selections chain via callbacks (e.g., `pickPerk` calls `showPassiveSelection` if pending).

Victory condition: all 10 attacks mastered (`DEFS.length` attacks at level 10).

### Key Helper Functions

- `nearest(S)` — closest enemy to player
- `dist2(ax,ay,bx,by)` — squared distance
- `dir(ax,ay,bx,by)` — normalized direction vector
- `tdm(S)` — total damage multiplier (`permDmgMult × dmgMult`)
- `critRoll(S)` — returns 1.2 on crit, else 1
- `proj(S, p)` — pushes a projectile onto `S.projs` with `tdm` applied
- `fx(S, e)` — pushes an effect onto `S.fxs`

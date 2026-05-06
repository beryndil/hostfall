# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

HOSTFALL — a retro single-screen sci-fi action game (browser-based). Four acts, ~15-20 minutes per playthrough. Full spec in `HOSTFALL_DESIGN.md.pdf`.

## Running the game

Open `index.html` in any modern browser. No build step, no dependencies, no package manager. The file is the game.

## Tech constraints — non-negotiable

- **Single file:** all code in `index.html` with embedded `<style>` and `<script>`. No external files.
- **Rendering:** HTML5 Canvas 2D. Internal resolution 320×240. Offscreen canvas at native res, blitted to a CSS-scaled display canvas (`image-rendering: pixelated`).
- **Sprites:** canvas primitives only. `drawSprite(spriteData, palette, x, y, scale)` paints 2D arrays of palette indices as small rects. No image files.
- **Audio:** Web Audio API only — oscillators, noise, envelopes. No audio files. Build an `Sfx` module.
- **Input:** keyboard only. WASD move, arrow keys aim/shoot, Space grenade, Shift sprint, E interact/pickup, Tab cycle weapon, R reload, Escape pause.
- **Persistence:** none. Death restarts current act. No localStorage.

## Code architecture

Organize the JS into 10 sections in this order, separated by `// =====` banners:

1. **Constants** — palette (16 colors), canvas dimensions, gameplay numbers
2. **Sprite data** — all sprite arrays (player ~12×16, enemies ~12×24, bosses ~32×48)
3. **Room data** — all 36 rooms as a single object/array (see Room object shape below)
4. **Audio system** — Web Audio init, `Sfx` functions, ambient drones
5. **Input system** — keyboard listeners, input state
6. **Entity classes** — `Player`, `Bullet`, `Enemy` (subclasses), `Boss`, `Pickup`, `Particle`, `Spawner`
7. **Room manager** — load, transition, clear-check
8. **Game loop** — `requestAnimationFrame` with fixed-timestep accumulator (1/60s); decouple update from render
9. **UI/HUD** — health pips, ammo, timer, menus, death/win screens
10. **Bootstrap** — init, canvas attach, start

**State machine** top-level states: `MENU`, `PLAYING`, `PAUSED`, `DYING`, `ACT_TRANSITION`, `WIN`, `LOSE`.

**Room object shape:**
```js
{
  id: "act1_r03",
  act: 1,
  walls: [{x, y, w, h}, ...],       // obstacle rects
  exits: { n: "act1_r04", s: null, e: "act1_r02", w: null },
  enemies: [{type, x, y}, ...],      // spawn list
  pickups: [{type, x, y}, ...],
  cleared: false,
  type: "combat" | "puzzle" | "story" | "transition" | "boss",
  lockExitsUntilCleared: true,
  bgColor: "#1a1a1a",
  wallStyle: "concrete" | "metal" | "organic" | "alien"
}
```

**Room transitions:** walk off edge with an exit → 100ms black flash → spawn 16px in from opposite edge of new room. Combat rooms lock exits (red barrier on doorways) until all enemies dead.

## Key mechanics (implement exactly as specced)

**Player:** 5 HP. 8×8 hitbox. Movement 90 px/s base. Sprint 1.5× for 2s, regen 4s. I-frames: 60 frames after hit (sprite flashes). Death: red flash + particles → fade black → "RESTARTING ACT" 1.5s → respawn at act start. Weapons persist through deaths within a run.

**Weapons:**
| # | Name | Dmg | Rate | Ammo | Notes |
|---|------|-----|------|------|-------|
| 1 | Pulse Rifle | 1 | 4/s | ∞ | default |
| 2 | Pistol | 1 | 6/s | ∞ | act 1 r2 |
| 3 | Plasma Rifle | 2 | 2/s | 30 (max 60) | pierces 1 enemy |
| 4 | Heavy Laser | 4 | 1/s | 8 (max 16) | instant beam, knockback |
| G | Grenade | 3 AoE | manual | 3 (max 5) | 0.8s fuse, 32px blast |

Projectile speeds: pulse 240 px/s, pistol 200, plasma 160. Tab cycles. Space throws grenade.

**Enemy AI:** All shooters intentionally miss — fire 8-16px off player. Pathfinding is direct line toward player, slide along walls on contact. No A*.

**Act 4 escape timer:** 3:00 countdown, prominent top-center. Screen shake intensifies below 1:30, more below 0:30. At 0:00: instant game over "TOO LATE", restart act 4.

## 16-color palette

```
0: #000000  1: #1a1a2e  2: #16213e  3: #3a2e1a
4: #7a7a7a  5: #c8c8c8  6: #5a3a1a  7: #8b6914
8: #2d5016  9: #7fb069  10: #d62828 11: #f77f00
12: #fcbf49 13: #7209b7 14: #f72585 15: #ffffff
```

Per-act background tinting: Act 1 neutral, Act 2 green/sickly, Act 3 deep purple/teal, Act 4 red emergency.

## Audio recipes (Web Audio API)

| Sound | Recipe |
|-------|--------|
| Pulse rifle shot | Square wave, 880→220 Hz over 0.06s, gain 0.3→0 |
| Pistol shot | Square, 1100→400 Hz over 0.04s |
| Plasma shot | Sawtooth, 220 Hz held 0.12s, low-pass filter sweeping down |
| Laser | Sine 880 Hz + saw 220 Hz, 0.3s, ring-mod feel |
| Grenade explosion | White noise burst, low-pass 800 Hz, 0.4s envelope |
| Enemy hit | Noise burst 0.05s, high-pass 2000 Hz |
| Enemy death | Noise burst 0.2s, low-pass sweeping down from 1500 |
| Player hit | Square 110 Hz, 0.15s, detune wobble |
| Pickup | Sine 660→990 Hz blip |
| Door unlock | Two-note rising blip |
| Boss phase transition | Low rumble (sine 60 Hz) + reverse cymbal (filtered noise rising) |

Ambient drones (instead of music): Act 1: sine 110 Hz + filtered noise. Act 2: detuned dual sine 90/93 Hz + high pings. Act 3: sine 60 Hz + slow LFO on cutoff. Act 4: pulsing siren (sine 440↔660 Hz square LFO) + low rumble. Default ambient at 30% of SFX volume.

## Build order

Each step must produce a runnable file before moving on:

1. Skeleton — HTML, canvas, game loop, palette, static room (4 walls + player rect)
2. Player movement — WASD + wall collision
3. Player shooting — arrow keys, bullets despawn off-screen
4. One enemy — Grunt patrol, shoots, dies in 2 hits, damages player, i-frames working
5. Room transitions — 3 connected rooms, exits/walls
6. Locked exits — combat rooms lock until cleared
7. HUD — health, ammo, weapon display
8. Audio system — shot/hit/death sounds
9. Build out act 1 — all 10 rooms, 3 enemy types, boss (both phases)
10. Pickups, weapon switching, grenades
11. Build out act 2 — new enemies, Brood Mother boss
12. Build out act 3 — new enemies, environmental hazards, Core Guardian
13. Act 4 — wave spawners, escape timer, Pursuer, end screen
14. Polish pass — particles, screen shake, ambient drones, transitions, menus

## Out of scope for v1

Do not build: save system, difficulty options, mouse/gamepad input, music (drones only), multiplayer, level editor, mobile/touch, externally loaded assets, cinematics/cutscenes, unlockables. Defer to v2.

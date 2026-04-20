# maczeta_life_gra — Design Spec
**Date:** 2026-04-19

## Overview

A new browser game combining the top-down 2.5D canvas renderer from **mlifegame** with the rich voice and sound system from **maczeta-life**. Adds a streak system with score multiplier, speed boost, and voice feedback.

---

## Architecture

- **New project:** `/Users/artm/Code/maczeta_life_gra/`
- **Base:** mlifegame's `index.html` (single-file, self-contained canvas game)
- **Approach:** Fork mlifegame. Remove its 3 basic Web Audio functions. Graft in maczeta-life's audio stack. Add streak state. Copy assets folder from maczeta-life.
- **Files:**
  - `index.html` — the entire game
  - `assets/` — copied from maczeta-life (voice MP3s + theme music)

---

## Sound System

**Remove** from mlifegame:
- `playSlashSound()`
- `playHitSound()`
- `playKillSound()`

**Add** from maczeta-life:
- `playSfx(type)` — layered Web Audio oscillator + noise for `hit`, `slash`, `kill`, `miss`
- `VOICE_LINES` object — 13 categories, each an array of MP3 paths:
  `start, hurt, kill, death, pickup, miss, streak, fury, finisher, weapon, lowhealth, dash, taunt`
- `playVoice(category)` — picks a random file from that category, plays it via `<audio>`, interrupts any currently playing voice line
- Background music: `assets/maczeta-life-theme.mp3`, loops, starts on game join at volume 0.24

**Voice triggers:**
| Event | Voice category |
|---|---|
| Game start | `start` |
| Player takes damage | `hurt` |
| Player kills enemy | `kill` (streak < 3), `streak` (streak 3–5), `fury` (streak 6+) |
| Player dies | `death` |
| Item picked up | `pickup` |
| Attack misses | `miss` |
| Attack misses (no hit) | `miss` |
| Low health (< 30) | `lowhealth` (max once per 10s) |

---

## Streak System

**State added to player:**
```
streakCount   — integer, kills chained within timeout window
streakTimer   — timestamp of last kill + 3000ms
```

**On each player kill:**
1. `streakCount++`
2. Score multiplier = `Math.min(1 + streakCount * 0.25, 3.0)` — score for that kill multiplied
3. Speed boost: `PLAYER_SPEED * 1.3` for 1200ms, then resets
4. Voice line selection:
   - streakCount === 1 → `kill`
   - streakCount 3–5 → `streak`
   - streakCount >= 6 → `fury`

**Streak reset:**
- If `gameTime > streakTimer` (3s without a kill): `streakCount = 0`, no voice
- On player death: `streakCount = 0`, play `death` voice

**HUD:**
- New streak badge below score: `🔥 x{streakCount}`, hidden when `streakCount < 2`
- Fades out (opacity transition) when streak resets

---

## What Stays Unchanged

- All of mlifegame's 2.5D top-down renderer (ground, buildings, trees, river, blood pools, particles, minimap)
- Bot AI and aggro system
- Item pickups (pierogi, złoty, oscypek, vodka, kielbasa, barszcz)
- Leaderboard and kill feed
- Mobile joystick + attack button controls
- Weapon: machete only (mlifegame already uses this exclusively)
- Map: Kraków landmarks, river layout, 3200×3200 world

---

## Out of Scope

- Multiple weapons (maczeta-life feature — not included)
- Fury bar / fury mode (maczeta-life feature — not included)
- Dash mechanic (not included)
- First-person / Three.js rendering (removed)

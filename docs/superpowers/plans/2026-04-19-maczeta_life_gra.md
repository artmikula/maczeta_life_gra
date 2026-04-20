# maczeta_life_gra Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a new top-down 2.5D browser game combining mlifegame's canvas renderer with maczeta-life's voice/sound system and a streak mechanic.

**Architecture:** Fork mlifegame's single `index.html`. Replace its 3 basic Web Audio functions with maczeta-life's richer `playSfx()` + `playVoice()` system. Add streak state (count, timer, speed boost, multiplier) to the kill handler. Copy maczeta-life's `assets/` folder for voice MP3s and music.

**Tech Stack:** Vanilla HTML/CSS/JS, Web Audio API, HTML `<audio>` for voice, Canvas 2D

---

### Task 1: Scaffold the project

**Files:**
- Create: `/Users/artm/Code/maczeta_life_gra/index.html` (copy from mlifegame)
- Create: `/Users/artm/Code/maczeta_life_gra/assets/` (copied from maczeta-life)

- [ ] **Step 1: Create project directory and copy base files**

```bash
mkdir -p /Users/artm/Code/maczeta_life_gra
cp /Users/artm/Code/mlifegame/index.html /Users/artm/Code/maczeta_life_gra/index.html
cp -r /Users/artm/Code/maczeta-life/assets /Users/artm/Code/maczeta_life_gra/assets
```

- [ ] **Step 2: Verify assets copied correctly**

```bash
ls /Users/artm/Code/maczeta_life_gra/assets/voice/ | wc -l
ls /Users/artm/Code/maczeta_life_gra/assets/
```

Expected: voice/ directory with 50+ mp3 files, and `maczeta-life-theme.mp3` present.

- [ ] **Step 3: Init git repo**

```bash
cd /Users/artm/Code/maczeta_life_gra
git init
git add index.html assets/
git commit -m "feat: scaffold maczeta_life_gra from mlifegame + maczeta-life assets"
```

---

### Task 2: Replace audio system

**Files:**
- Modify: `/Users/artm/Code/maczeta_life_gra/index.html` — replace sound section (~lines 416–484)

- [ ] **Step 1: Replace the three old audio functions with maczeta-life's audio stack**

In `index.html`, find this block and replace it entirely:

```js
// ========== SOUND (Web Audio) ==========
let audioCtx = null;
function ensureAudio() { if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)(); return audioCtx; }

function playSlashSound() {
  ...
}

function playHitSound() {
  ...
}

function playKillSound() {
  ...
}
```

Replace with:

```js
// ========== SOUND ==========
let audioCtx = null;
function getAudioCtx() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  return audioCtx;
}

function playSfx(type) {
  const ctx = getAudioCtx();
  if (ctx.state === 'suspended') ctx.resume();
  const now = ctx.currentTime;

  if (type === 'hit') {
    const noise = ctx.createBufferSource();
    const buf = ctx.createBuffer(1, ctx.sampleRate * 0.12, ctx.sampleRate);
    const data = buf.getChannelData(0);
    for (let i = 0; i < data.length; i++) data[i] = (Math.random() * 2 - 1) * Math.exp(-i / (data.length * 0.15));
    noise.buffer = buf;
    const filter = ctx.createBiquadFilter();
    filter.type = 'lowpass';
    filter.frequency.setValueAtTime(1800, now);
    filter.frequency.exponentialRampToValueAtTime(200, now + 0.1);
    const gain = ctx.createGain();
    gain.gain.setValueAtTime(0.7, now);
    gain.gain.exponentialRampToValueAtTime(0.01, now + 0.12);
    noise.connect(filter).connect(gain).connect(ctx.destination);
    noise.start(now);
    const thud = ctx.createOscillator();
    thud.type = 'sine';
    thud.frequency.setValueAtTime(90, now);
    thud.frequency.exponentialRampToValueAtTime(30, now + 0.08);
    const thudGain = ctx.createGain();
    thudGain.gain.setValueAtTime(0.6, now);
    thudGain.gain.exponentialRampToValueAtTime(0.01, now + 0.08);
    thud.connect(thudGain).connect(ctx.destination);
    thud.start(now); thud.stop(now + 0.1);
  }

  if (type === 'slash') {
    const noise = ctx.createBufferSource();
    const buf = ctx.createBuffer(1, ctx.sampleRate * 0.15, ctx.sampleRate);
    const data = buf.getChannelData(0);
    for (let i = 0; i < data.length; i++) data[i] = (Math.random() * 2 - 1) * Math.sin(i / data.length * Math.PI) * 0.4;
    noise.buffer = buf;
    const filter = ctx.createBiquadFilter();
    filter.type = 'bandpass';
    filter.frequency.setValueAtTime(3000, now);
    filter.frequency.exponentialRampToValueAtTime(800, now + 0.12);
    filter.Q.value = 2;
    const gain = ctx.createGain();
    gain.gain.setValueAtTime(0.35, now);
    gain.gain.exponentialRampToValueAtTime(0.01, now + 0.15);
    noise.connect(filter).connect(gain).connect(ctx.destination);
    noise.start(now);
  }

  if (type === 'kill') {
    playSfx('hit');
    const crunch = ctx.createBufferSource();
    const buf = ctx.createBuffer(1, ctx.sampleRate * 0.25, ctx.sampleRate);
    const data = buf.getChannelData(0);
    for (let i = 0; i < data.length; i++) {
      const t = i / data.length;
      data[i] = (Math.random() * 2 - 1) * Math.exp(-t * 6) * (1 + Math.sin(t * 400) * 0.3);
    }
    crunch.buffer = buf;
    const filter = ctx.createBiquadFilter();
    filter.type = 'lowpass';
    filter.frequency.setValueAtTime(2400, now);
    filter.frequency.exponentialRampToValueAtTime(100, now + 0.2);
    const gain = ctx.createGain();
    gain.gain.setValueAtTime(0.5, now + 0.03);
    gain.gain.exponentialRampToValueAtTime(0.01, now + 0.25);
    crunch.connect(filter).connect(gain).connect(ctx.destination);
    crunch.start(now + 0.03);
    const boom = ctx.createOscillator();
    boom.type = 'sine';
    boom.frequency.setValueAtTime(60, now);
    boom.frequency.exponentialRampToValueAtTime(20, now + 0.18);
    const boomGain = ctx.createGain();
    boomGain.gain.setValueAtTime(0.8, now);
    boomGain.gain.exponentialRampToValueAtTime(0.01, now + 0.2);
    boom.connect(boomGain).connect(ctx.destination);
    boom.start(now); boom.stop(now + 0.22);
  }

  if (type === 'miss') {
    const noise = ctx.createBufferSource();
    const buf = ctx.createBuffer(1, ctx.sampleRate * 0.2, ctx.sampleRate);
    const data = buf.getChannelData(0);
    for (let i = 0; i < data.length; i++) data[i] = (Math.random() * 2 - 1) * Math.sin(i / data.length * Math.PI) * 0.25;
    noise.buffer = buf;
    const filter = ctx.createBiquadFilter();
    filter.type = 'highpass';
    filter.frequency.value = 2000;
    const gain = ctx.createGain();
    gain.gain.setValueAtTime(0.3, now);
    gain.gain.exponentialRampToValueAtTime(0.01, now + 0.18);
    noise.connect(filter).connect(gain).connect(ctx.destination);
    noise.start(now);
  }
}

const VOICE_LINES = {
  start:     ['./assets/voice/start-1.mp3','./assets/voice/start-2.mp3','./assets/voice/start-3.mp3'],
  hurt:      ['./assets/voice/hurt-1.mp3','./assets/voice/hurt-2.mp3','./assets/voice/hurt-3.mp3','./assets/voice/hurt-4.mp3'],
  kill:      ['./assets/voice/kill-1.mp3','./assets/voice/kill-2.mp3','./assets/voice/kill-3.mp3','./assets/voice/kill-4.mp3','./assets/voice/kill-5.mp3','./assets/voice/kill-6.mp3','./assets/voice/kill-7.mp3','./assets/voice/kill-8.mp3','./assets/voice/kill-9.mp3','./assets/voice/kill-10.mp3'],
  death:     ['./assets/voice/death-1.mp3','./assets/voice/death-2.mp3','./assets/voice/death-3.mp3'],
  pickup:    ['./assets/voice/pickup-1.mp3','./assets/voice/pickup-2.mp3'],
  miss:      ['./assets/voice/miss-1.mp3','./assets/voice/miss-2.mp3','./assets/voice/miss-3.mp3'],
  streak:    ['./assets/voice/streak-1.mp3','./assets/voice/streak-2.mp3','./assets/voice/streak-3.mp3','./assets/voice/streak-4.mp3','./assets/voice/streak-5.mp3'],
  fury:      ['./assets/voice/fury-1.mp3','./assets/voice/fury-2.mp3','./assets/voice/fury-3.mp3'],
  lowhealth: ['./assets/voice/lowhealth-1.mp3','./assets/voice/lowhealth-2.mp3','./assets/voice/lowhealth-3.mp3'],
  taunt:     ['./assets/voice/taunt-1.mp3','./assets/voice/taunt-2.mp3','./assets/voice/taunt-3.mp3','./assets/voice/taunt-4.mp3','./assets/voice/taunt-5.mp3'],
};

let currentVoice = null;
function playVoice(category) {
  const lines = VOICE_LINES[category];
  if (!lines || !lines.length) return;
  const src = lines[Math.floor(Math.random() * lines.length)];
  if (currentVoice) { currentVoice.pause(); currentVoice.currentTime = 0; }
  currentVoice = new Audio(src);
  currentVoice.volume = 0.85;
  currentVoice.play().catch(() => {});
}

const music = new Audio('./assets/maczeta-life-theme.mp3');
music.loop = true;
music.volume = 0.24;
```

- [ ] **Step 2: Update the two call sites of the old slash function**

Find:
```js
  addSlashParticle(p.x + Math.cos(p.angle) * 30, p.y + Math.sin(p.angle) * 30, p.angle);
  playSlashSound();
```
(appears twice — once in `mousedown`, once in `touchstart`)

Replace both with:
```js
  addSlashParticle(p.x + Math.cos(p.angle) * 30, p.y + Math.sin(p.angle) * 30, p.angle);
  playSfx('slash');
```

- [ ] **Step 3: Update hit sound call site**

Find (inside hit detection, after `screenShake = 5;`):
```js
          playHitSound();
```
Replace with:
```js
          playSfx('hit');
```

- [ ] **Step 4: Update kill sound call site**

Find (inside kill block, after `screenShake = 10;`):
```js
            playKillSound();
```
Replace with:
```js
            playSfx('kill');
```

- [ ] **Step 5: Open in browser and verify sounds work**

Open `/Users/artm/Code/maczeta_life_gra/index.html` in a browser (or via `open index.html`).
- Join the game
- Attack an enemy — should hear a bandpass swoosh (slash)
- Hit an enemy — should hear a low thud + noise burst (hit)
- Kill an enemy — should hear a deep crunch + boom (kill)
- No console errors about undefined functions

- [ ] **Step 6: Commit**

```bash
cd /Users/artm/Code/maczeta_life_gra
git add index.html
git commit -m "feat: replace audio with maczeta-life sfx system + voice infrastructure"
```

---

### Task 3: Wire voice lines to game events

**Files:**
- Modify: `/Users/artm/Code/maczeta_life_gra/index.html`

- [ ] **Step 1: Add streak HUD element and bind it in JS** *(must come before voice wiring — death handler uses it)*

In HTML, find:
```html
  <div class="stat" id="hud-combo" style="display:none;border-left-color:#f4a261">🔥 x1</div>
```
Add after it:
```html
  <div class="stat" id="hud-streak" style="display:none;border-left-color:#f4a261;transition:opacity 0.4s">🔥 x1</div>
```

In JS, find:
```js
const hudCombo = document.getElementById('hud-combo');
```
Add after it:
```js
const hudStreak = document.getElementById('hud-streak');
```

- [ ] **Step 2: Add streak/lowhealth state variables to the STATE section**

Find this block in the STATE section (around `let comboCount = 0;`):
```js
let comboCount = 0;
let comboTimer = 0;
```

Add after it:
```js
let streakCount = 0;
let streakTimer = 0;
let streakSpeedUntil = 0;
let lastLowHealthVoice = 0;
```

- [ ] **Step 2: Play music and start voice on game join**

Find in `startGame()`:
```js
  playing = true;
  joinScreen.style.display = 'none';
```

Add before that line:
```js
  music.currentTime = 0;
  music.play().catch(() => {});
  setTimeout(() => playVoice('start'), 300);
  streakCount = 0; streakTimer = 0; streakSpeedUntil = 0;
  lastLowHealthVoice = 0;
```

- [ ] **Step 3: Play hurt voice when player takes damage**

Find inside the hit detection block (after `playSfx('hit');`):

```js
          playSfx('hit');

          if (o.health <= 0) {
```

Replace with:

```js
          playSfx('hit');
          if (o === me) playVoice('hurt');

          if (o.health <= 0) {
```

- [ ] **Step 4: Play death voice when player dies**

Find:
```js
            if (o === me) {
              deathScreen.style.display = 'flex';
              deathMsgEl.textContent = msg;
            }
```

Replace with:
```js
            if (o === me) {
              deathScreen.style.display = 'flex';
              deathMsgEl.textContent = msg;
              playVoice('death');
              streakCount = 0; streakTimer = 0; streakSpeedUntil = 0;
              hudStreak.style.display = 'none';
            }
```

(Note: `hudStreak` is added in Task 4 — make sure Task 4 runs before testing this step.)

- [ ] **Step 5: Play pickup voice for player item pickups**

Find:
```js
        if (e === me) addFloatingText(it.x, it.y - 20, `+${it.pts}`, it.glow);
        items.splice(i, 1);
```

Replace with:
```js
        if (e === me) {
          addFloatingText(it.x, it.y - 20, `+${it.pts}`, it.glow);
          if (Math.random() < 0.5) playVoice('pickup');
        }
        items.splice(i, 1);
```

- [ ] **Step 6: Add miss voice and low health voice check in update()**

Find at the end of the update function, just before `// Particles`:
```js
  // Combo timeout
  if (comboTimer > 0 && gameTime > comboTimer) {
    comboCount = 0; comboTimer = 0;
    hudCombo.style.display = 'none';
  }

  // Particles
```

Add between those two blocks:
```js
  // Miss voice: player attacked but hit no one this frame
  if (me && me.alive && me.attacking && gameTime - me.attackStart < 20) {
    const hitAny = entities.some(o => o !== me && o.alive && o.hitFlash > 6);
    if (!hitAny && Math.random() < 0.25) playVoice('miss');
  }

  // Low health voice (throttled to once per 10s)
  if (me && me.alive && me.health < 30 && gameTime - lastLowHealthVoice > 10000) {
    lastLowHealthVoice = gameTime;
    playVoice('lowhealth');
  }

```

- [ ] **Step 7: Open in browser and verify voice events**

Open the game, join, and check:
- A voice line plays shortly after joining (start)
- Taking a hit triggers a hurt voice
- Picking up an item sometimes triggers a pickup voice
- Low health (get hit a lot) triggers a lowhealth voice eventually
- Music plays in the background
- No console errors

- [ ] **Step 8: Commit**

```bash
cd /Users/artm/Code/maczeta_life_gra
git add index.html
git commit -m "feat: wire voice lines to game events (start, hurt, death, pickup, miss, lowhealth)"
```

---

### Task 4: Streak system — kill logic, multiplier, speed boost

**Files:**
- Modify: `/Users/artm/Code/maczeta_life_gra/index.html`

- [ ] **Step 1: Replace the kill handler for player kills with streak logic**

Find the player kill block:
```js
            // Combo system
            if (e === me) {
              comboCount++;
              comboTimer = gameTime + 3000;
              const bonus = comboCount > 1 ? comboCount * 25 : 0;
              e.score += 100 + bonus;
              e.kills++;

              if (comboCount > 1) {
                comboText.textContent = `🔥 x${comboCount} COMBO! +${bonus}`;
                comboDisplay.style.display = 'block';
                setTimeout(() => { comboDisplay.style.display = 'none'; }, 1500);
                hudCombo.style.display = 'block';
                hudCombo.textContent = `🔥 x${comboCount}`;
              }
            } else {
              e.score += 100;
              e.kills++;
            }
```

Replace with:
```js
            // Streak system
            if (e === me) {
              // Streak: reset if timed out, else increment
              if (gameTime > streakTimer) streakCount = 0;
              streakCount++;
              streakTimer = gameTime + 3000;

              // Score multiplier: 1.0 at streak 1, up to 3.0
              const multiplier = Math.min(1 + (streakCount - 1) * 0.25, 3.0);
              const baseScore = 100;
              const earnedScore = Math.round(baseScore * multiplier);
              e.score += earnedScore;
              e.kills++;

              // Speed boost for 1.2s
              streakSpeedUntil = gameTime + 1200;

              // Combo display
              if (streakCount > 1) {
                comboText.textContent = `🔥 x${streakCount}  +${earnedScore}`;
                comboDisplay.style.display = 'block';
                setTimeout(() => { comboDisplay.style.display = 'none'; }, 1500);
              }

              // Streak HUD badge
              if (streakCount >= 2) {
                hudStreak.style.display = 'block';
                hudStreak.style.opacity = '1';
                hudStreak.textContent = `🔥 x${streakCount}`;
              }

              // Voice line based on streak tier
              if (streakCount >= 6) {
                playVoice('fury');
              } else if (streakCount >= 3) {
                playVoice('streak');
              } else {
                playVoice('kill');
              }
            } else {
              e.score += 100;
              e.kills++;
            }
```

- [ ] **Step 2: Apply speed boost to player movement**

Find in the player movement block:
```js
      const nx = me.x + (dx / len) * PLAYER_SPEED;
      const ny = me.y + (dy / len) * PLAYER_SPEED;
```

Replace with:
```js
      const curSpeed = gameTime < streakSpeedUntil ? PLAYER_SPEED * 1.3 : PLAYER_SPEED;
      const nx = me.x + (dx / len) * curSpeed;
      const ny = me.y + (dy / len) * curSpeed;
```

- [ ] **Step 3: Reset streak HUD on timeout**

Find:
```js
  // Combo timeout
  if (comboTimer > 0 && gameTime > comboTimer) {
    comboCount = 0; comboTimer = 0;
    hudCombo.style.display = 'none';
  }
```

Add after it:
```js
  // Streak timeout
  if (streakCount > 0 && gameTime > streakTimer) {
    streakCount = 0;
    hudStreak.style.opacity = '0';
    setTimeout(() => { hudStreak.style.display = 'none'; }, 400);
  }
```

- [ ] **Step 4: Open in browser and verify streak**

Open the game and:
- Kill one enemy → hear `kill` voice, no badge visible
- Kill a second enemy within 3s → badge shows `🔥 x2`, combo popup appears, hear `kill` voice, movement feels faster briefly
- Kill a 3rd within 3s → hear `streak` voice
- Kill 6+ in a chain → hear `fury` voice, badge shows high number
- Wait 3s without killing → badge fades out
- Die → badge disappears, streak resets

- [ ] **Step 5: Commit**

```bash
cd /Users/artm/Code/maczeta_life_gra
git add index.html
git commit -m "feat: add streak system with multiplier, speed boost, and tiered voice lines"
```

---

### Task 5: Final polish and self-review

**Files:**
- Modify: `/Users/artm/Code/maczeta_life_gra/index.html`

- [ ] **Step 1: Update the page title**

Find:
```html
<title>Maczeta Life — Free .io Machete Game Set in Kraków, Poland</title>
```

Replace with:
```html
<title>Maczeta Life Gra — Top-Down Machete Game in Kraków</title>
```

- [ ] **Step 2: Full playthrough verification**

Open the game in a browser and run through this checklist:
- [ ] Join screen loads, fake player count shows
- [ ] Clicking Join starts the game and music plays
- [ ] Start voice fires ~300ms after joining
- [ ] WASD moves the player around the map
- [ ] Clicking attacks with a swoosh sound
- [ ] Hitting an enemy plays a thud
- [ ] Taking a hit plays a hurt voice
- [ ] Killing an enemy plays kill voice + playSfx('kill')
- [ ] Streak badge appears at x2, voice changes at x3 and x6
- [ ] Speed boost feels noticeable after each kill
- [ ] Badge fades after 3s idle
- [ ] Picking up pierogi sometimes plays pickup voice
- [ ] Dying plays death voice and resets streak badge
- [ ] After respawn, game continues normally
- [ ] Music loops continuously
- [ ] No console errors throughout

- [ ] **Step 3: Final commit**

```bash
cd /Users/artm/Code/maczeta_life_gra
git add index.html
git commit -m "feat: polish title and verify full playthrough"
```

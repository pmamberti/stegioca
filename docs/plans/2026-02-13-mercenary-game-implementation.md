# Mercenary Mission Game Prototype — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a playable browser game where you pick 2 of 6 procedurally generated mercenaries and send them on a mission with a narrative outcome.

**Architecture:** Single `index.html` with three inline layers — generation (pure functions producing character data and pixel grids), rendering (Canvas 2D drawing scaled pixel art), and game logic (state management and screen transitions). No dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS, Canvas 2D API, single file.

**Design Doc:** `docs/plans/2026-02-13-mercenary-game-prototype-design.md`

---

### Task 1: HTML Scaffold & CSS Layout

**Files:**
- Create: `index.html`

**Step 1: Write the HTML structure**

Create `index.html` with three screen `<div>`s (selection, waiting, result), each hidden by default. Include:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Stegioca</title>
  <style>/* Step 2 */</style>
</head>
<body>
  <div id="screen-selection" class="screen active">
    <h1>CHOOSE YOUR TEAM</h1>
    <div id="card-grid" class="card-grid">
      <!-- 6 cards injected by JS -->
    </div>
    <div class="bottom-bar">
      <span id="selection-counter">0 / 2 selected</span>
      <button id="btn-start" disabled>START MISSION</button>
    </div>
  </div>

  <div id="screen-waiting" class="screen">
    <h1>MISSION IN PROGRESS</h1>
    <div id="waiting-cards" class="waiting-cards">
      <!-- 2 selected cards shown here -->
    </div>
    <p id="waiting-text" class="blink">...</p>
  </div>

  <div id="screen-result" class="screen">
    <h2 id="result-heading"></h2>
    <div id="result-cards" class="waiting-cards"></div>
    <p id="result-narrative"></p>
    <button id="btn-new-mission">NEW MISSION</button>
  </div>

  <script>/* Tasks 2-8 */</script>
</body>
</html>
```

**Step 2: Write the CSS**

1-bit aesthetic: black background, white text, monospace font. Card grid is 3x2 CSS Grid. Cards have white 1px borders. Selected cards get a thicker/inverted border. Screens swap via `.active` class. Keep it minimal.

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  background: #000;
  color: #fff;
  font-family: 'Courier New', monospace;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
.screen { display: none; flex-direction: column; align-items: center; gap: 20px; width: 100%; max-width: 700px; padding: 20px; }
.screen.active { display: flex; }
h1, h2 { text-align: center; letter-spacing: 2px; }
.card-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; width: 100%; }
.card {
  border: 2px solid #444;
  padding: 12px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  transition: border-color 0.15s;
}
.card:hover { border-color: #888; }
.card.selected { border-color: #fff; border-width: 3px; }
.card canvas { image-rendering: pixelated; width: 128px; height: 128px; }
.card .name { font-size: 14px; font-weight: bold; }
.card .race-tag { font-size: 11px; padding: 2px 6px; border: 1px solid #666; text-transform: uppercase; letter-spacing: 1px; }
.bottom-bar { display: flex; justify-content: space-between; align-items: center; width: 100%; margin-top: 16px; }
button {
  background: #fff;
  color: #000;
  border: none;
  padding: 10px 24px;
  font-family: inherit;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  letter-spacing: 1px;
  text-transform: uppercase;
}
button:disabled { opacity: 0.3; cursor: not-allowed; }
.waiting-cards { display: flex; gap: 24px; justify-content: center; }
.blink { animation: blink 1s step-end infinite; }
@keyframes blink { 50% { opacity: 0; } }
#result-heading { font-size: 28px; }
#result-narrative { max-width: 500px; text-align: center; line-height: 1.6; }
```

**Step 3: Verify**

Open `index.html` in a browser. You should see:
- Black screen with "CHOOSE YOUR TEAM" heading
- Empty card grid area
- "0 / 2 selected" counter and disabled "START MISSION" button
- Other screens hidden

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add HTML scaffold and 1-bit CSS layout"
```

---

### Task 2: Seeded Random Number Generator

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement seeded RNG**

A simple mulberry32 PRNG. This is the foundation — every procedural feature uses it so results are reproducible per seed.

```javascript
// ── Seeded RNG (mulberry32) ──
function createRNG(seed) {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}
```

**Step 2: Verify**

Add a temporary `console.log` that creates an RNG with seed 42 and logs 5 values. Open browser console. Same 5 values every refresh = working.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add seeded RNG (mulberry32)"
```

---

### Task 3: Name Generation System

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement syllable-based name generator**

Define syllable pools per race. Generator picks 2-3 syllables using the seeded RNG, capitalizes.

```javascript
// ── Name Generation ──
const NAME_SYLLABLES = {
  Human:     { first: ['dev','en','dra','mar','co','li','sa','jon','ray','el','ka','ri'],
               last:  ['ray','sol','kirk','vale','cross','stone','ward','finn'] },
  Tharn:     { first: ['grok','thur','brak','gor','kru','dag','vol','ruk','mog','zar'],
               last:  ['nash','gul','dok','mar','thok','gar','bur'] },
  Cephalid:  { first: ['zyl','thu','pho','gla','cthu','nyl','rhy','vos','qua','mur'],
               last:  ['ath','oth','ull','yth','esh','arn','ix'] },
  Gravborn:  { first: ['bor','ked','sten','dol','grun','tor','hal','vik','mag','rek'],
               last:  ['ax','ven','dig','holm','pit','ore','shaft'] },
  Insectoid: { first: ['zik','chi','kri','bzz','tik','xer','cli','vri','ski','phi'],
               last:  ['xis','tak','rik','zzt','kis','thi','nik'] },
  Construct: { first: ['MK','AX','SR','TX','VK','NX','RX','BX','CX','ZR'],
               last:  ['7','13','99','42','08','31','66','0','77','51'] },
};

function generateName(race, rng) {
  const s = NAME_SYLLABLES[race];
  const first = s.first[Math.floor(rng() * s.first.length)];
  const last = s.last[Math.floor(rng() * s.last.length)];
  if (race === 'Construct') return first + '-' + last;
  return first.charAt(0).toUpperCase() + first.slice(1) + ' ' + last.charAt(0).toUpperCase() + last.slice(1);
}
```

**Step 2: Verify**

Temporary `console.log` generating 3 names per race. Check: names look appropriate for each race, same seed = same names.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add syllable-based name generator for 6 races"
```

---

### Task 4: Race Portrait Templates

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Define base templates**

Each race gets a 16x32 half-template (left side only — will be mirrored). Stored as arrays of strings where `#` = filled pixel, `.` = empty. 16 wide because we mirror to get 32. The template defines the silhouette; the generation step adds variation.

Define all 6 race templates. Key visual differentiators:
- **Human** — round head, normal shoulders, straight body
- **Tharn** — wide cranial ridges, massive shoulders, thick neck
- **Cephalid** — bulbous dome head, thin neck, tendrils hanging down from face
- **Gravborn** — short stature (starts lower), very wide, visor band across face
- **Insectoid** — antennae protruding up, narrow head, segmented thorax/abdomen
- **Construct** — flat-top head, perfectly rectangular shoulders, angular body

```javascript
// ── Race Templates (16 wide x 32 tall, left half only) ──
// '#' = always filled, '.' = always empty, 'x' = variable (seed-driven)
const RACE_TEMPLATES = {
  Human: [
    '................',
    '................',
    '......####......',
    '.....######.....',
    '....########....',
    '....###xx###....',
    '....########....',
    '....##x..x##....',  // eyes
    '....###..###....',
    '....##.xx.##....',  // mouth
    '.....######.....',
    '......####......',  // chin
    '.......##.......',  // neck
    '......####......',
    '.....######.....',
    '....###xx###....',  // shoulders
    '....########....',
    '....########....',
    '.....######.....',
    '.....######.....',
    '.....##xx##.....',  // belt area
    '.....######.....',
    '.....######.....',
    '.....#....#.....',  // legs split
    '....##....##....',
    '....##....##....',
    '....##....##....',
    '....##....##....',
    '....###..###....',  // boots
    '................',
    '................',
    '................',
  ],
  // ... (similar 32-row arrays for each race)
};
```

Write the full template for all 6 races with visually distinct silhouettes. Each template is 32 rows of 16 characters.

**Step 2: Verify**

Temporary: render one template to a canvas by iterating the string array. Open browser, confirm the silhouette looks like the intended race.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add 6 race portrait base templates"
```

---

### Task 5: Portrait Generation with Seed Variation

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement portrait generator**

Takes a race template and seed, produces a 32x32 Uint8Array:

1. Parse the template rows
2. For each `x` cell, use seeded RNG to decide filled or empty (adds variation)
3. Mirror left half to right half to produce full 32x32 grid
4. Return the Uint8Array

```javascript
// ── Portrait Generation ──
function generatePortrait(race, rng) {
  const template = RACE_TEMPLATES[race];
  const size = 32;
  const pixels = new Uint8Array(size * size);

  for (let y = 0; y < size; y++) {
    const row = template[y];
    for (let x = 0; x < size / 2; x++) {
      const ch = row[x];
      let filled = 0;
      if (ch === '#') filled = 1;
      else if (ch === 'x') filled = rng() > 0.5 ? 1 : 0;
      // else '.' = 0

      // Set left side
      pixels[y * size + x] = filled;
      // Mirror to right side
      pixels[y * size + (size - 1 - x)] = filled;
    }
  }

  return pixels;
}
```

**Step 2: Implement full character generator**

Combines RNG, name, race, and portrait into one character object.

```javascript
function generateCharacter(race, seed) {
  const rng = createRNG(seed);
  return {
    name: generateName(race, rng),
    race: race,
    portrait: generatePortrait(race, rng),
    seed: seed,
  };
}
```

**Step 3: Verify**

Generate two characters of same race with different seeds. Log both. Confirm names differ and portraits differ. Generate same seed twice — confirm identical output.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add portrait generation with seed-driven variation and symmetry"
```

---

### Task 6: Card Rendering

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement card renderer**

Function that takes a character object and returns a `.card` DOM element with a canvas portrait, name, and race tag.

```javascript
// ── Card Rendering ──
function renderCard(character) {
  const card = document.createElement('div');
  card.className = 'card';
  card.dataset.seed = character.seed;

  // Portrait canvas
  const canvas = document.createElement('canvas');
  canvas.width = 32;
  canvas.height = 32;
  const ctx = canvas.getContext('2d');
  const imageData = ctx.createImageData(32, 32);

  for (let i = 0; i < character.portrait.length; i++) {
    const v = character.portrait[i] ? 255 : 0;
    const idx = i * 4;
    imageData.data[idx] = v;     // R
    imageData.data[idx + 1] = v; // G
    imageData.data[idx + 2] = v; // B
    imageData.data[idx + 3] = 255; // A
  }
  ctx.putImageData(imageData, 0, 0);

  // Name label
  const name = document.createElement('div');
  name.className = 'name';
  name.textContent = character.name;

  // Race tag
  const tag = document.createElement('div');
  tag.className = 'race-tag';
  tag.textContent = character.race;

  card.appendChild(canvas);
  card.appendChild(name);
  card.appendChild(tag);

  return card;
}
```

**Step 2: Wire up initial render**

Generate 6 characters (one per race, random seeds) and render them into the card grid on page load.

```javascript
const RACES = ['Human', 'Tharn', 'Cephalid', 'Gravborn', 'Insectoid', 'Construct'];

function generateRoster() {
  return RACES.map(race => generateCharacter(race, Math.floor(Math.random() * 100000)));
}

let roster = generateRoster();

function renderSelectionScreen() {
  const grid = document.getElementById('card-grid');
  grid.innerHTML = '';
  roster.forEach(char => {
    grid.appendChild(renderCard(char));
  });
}

renderSelectionScreen();
```

**Step 3: Verify**

Open browser. You should see 6 cards in a 3x2 grid, each with a distinct pixel art silhouette, a name, and a race tag. Refresh — new names and slight portrait variations (different seeds).

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: render 6 procedurally generated mercenary cards"
```

---

### Task 7: Selection Logic & Screen Transitions

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement card selection**

Track selected cards (max 2). Toggle on click. Update counter and button state.

```javascript
// ── Game State ──
let selectedIndices = [];

function updateSelectionUI() {
  const counter = document.getElementById('selection-counter');
  const btn = document.getElementById('btn-start');
  counter.textContent = `${selectedIndices.length} / 2 selected`;
  btn.disabled = selectedIndices.length !== 2;

  document.querySelectorAll('.card').forEach((card, i) => {
    card.classList.toggle('selected', selectedIndices.includes(i));
  });
}

function handleCardClick(index) {
  const pos = selectedIndices.indexOf(index);
  if (pos !== -1) {
    selectedIndices.splice(pos, 1);
  } else if (selectedIndices.length < 2) {
    selectedIndices.push(index);
  }
  updateSelectionUI();
}
```

Add click listeners when rendering cards (modify `renderSelectionScreen`):

```javascript
// Inside renderSelectionScreen, after creating card element:
card.addEventListener('click', () => handleCardClick(i));
// where i is the index from the forEach
```

**Step 2: Implement screen switching**

```javascript
function showScreen(screenId) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.getElementById(screenId).classList.add('active');
}
```

**Step 3: Wire up "Start Mission" button**

```javascript
document.getElementById('btn-start').addEventListener('click', () => {
  startMission();
});

function startMission() {
  const selected = selectedIndices.map(i => roster[i]);
  showScreen('screen-waiting');
  renderWaitingScreen(selected);
  setTimeout(() => showResult(selected), 5000);
}
```

**Step 4: Verify**

Open browser. Click cards — they highlight, counter updates. Third click ignored when 2 selected. Click selected card to deselect. Button enables at exactly 2. Click button — switches to waiting screen.

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add card selection logic and screen transitions"
```

---

### Task 8: Waiting Screen, Result Screen & Narratives

**Files:**
- Modify: `index.html` (inside `<script>` tag)

**Step 1: Implement waiting screen renderer**

```javascript
function renderWaitingScreen(selected) {
  const container = document.getElementById('waiting-cards');
  container.innerHTML = '';
  selected.forEach(char => container.appendChild(renderCard(char)));
}
```

**Step 2: Implement narrative pools**

```javascript
const NARRATIVES = {
  success: [
    "{name1} drew their fire while {name2} secured the objective. Clean extraction.",
    "{name2} almost blew the cover, but {name1} improvised. Mission complete.",
    "Textbook operation. {name1} and {name2} made it look easy.",
    "{name1} cracked the encryption. {name2} held the corridor. Data secured.",
    "Against the odds, {name2} found an alternate route. {name1} covered the exit.",
    "{name1} and {name2} moved like they'd trained together for years. Flawless.",
    "The target never saw {name2} coming. {name1} handled the getaway.",
    "{name1} took point. {name2} watched their six. Not a scratch on either.",
    "Messy start, clean finish. {name2} adapted fast. {name1} kept their nerve.",
    "{name1} disabled the alarms. {name2} grabbed the payload. In and out.",
  ],
  failure: [
    "{name1} took a hit in the first corridor. {name2} dragged them out. No payload.",
    "Bad intel. {name1} and {name2} walked into a trap. Barely made it back.",
    "{name2} froze under fire. {name1} called the abort. Live to fight another day.",
    "The lock was military-grade. {name1} couldn't crack it. {name2} burned through their ammo covering.",
    "{name1} and {name2} got separated in the smoke. Extraction was ugly.",
    "Target was already gone. {name1} and {name2} spent three hours in a dead facility.",
    "{name2} tripped a silent alarm. By the time {name1} noticed, it was too late.",
    "Comms went dark ten minutes in. {name1} and {name2} had to improvise an exit.",
    "The briefing said light resistance. It wasn't. {name2} got {name1} out, but that's all.",
    "{name1} made the call to pull out. {name2} disagreed, but they're both alive.",
  ],
};
```

**Step 3: Implement result screen**

```javascript
function showResult(selected) {
  const success = Math.random() < 0.5;
  const pool = success ? NARRATIVES.success : NARRATIVES.failure;
  const template = pool[Math.floor(Math.random() * pool.length)];
  const narrative = template
    .replace(/\{name1\}/g, selected[0].name)
    .replace(/\{name2\}/g, selected[1].name);

  document.getElementById('result-heading').textContent = success ? 'MISSION SUCCESS' : 'MISSION FAILED';
  document.getElementById('result-heading').style.color = success ? '#fff' : '#888';
  document.getElementById('result-narrative').textContent = narrative;

  const container = document.getElementById('result-cards');
  container.innerHTML = '';
  selected.forEach(char => container.appendChild(renderCard(char)));

  showScreen('screen-result');
}
```

**Step 4: Wire up "New Mission" button**

```javascript
document.getElementById('btn-new-mission').addEventListener('click', () => {
  selectedIndices = [];
  roster = generateRoster();
  renderSelectionScreen();
  updateSelectionUI();
  showScreen('screen-selection');
});
```

**Step 5: Verify — full playthrough**

1. Open browser — see 6 cards
2. Select 2 — counter updates, button enables
3. Click "Start Mission" — waiting screen with 2 cards, blinking dots
4. After 5 seconds — result screen with heading, narrative with character names, cards visible
5. Click "New Mission" — back to selection with 6 fresh characters
6. Repeat — different names, portraits, narratives each time

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add mission flow with waiting screen and narrative outcomes"
```

---

### Task 9: Final Polish & Push

**Files:**
- Modify: `index.html`

**Step 1: Remove any temporary console.logs**

Clean up any debugging code left from earlier tasks.

**Step 2: Add a page title/header**

A small "STEGIOCA" title at the top of the selection screen, styled in the 1-bit aesthetic.

**Step 3: Test edge cases**

- Rapidly clicking cards
- Clicking "Start Mission" multiple times
- Browser back/forward (should be harmless since it's all one page)

**Step 4: Final commit and push**

```bash
git add index.html
git commit -m "feat: polish UI and clean up prototype"
git push origin main
```

---

## Summary

| Task | What | ~Time |
|------|------|-------|
| 1 | HTML scaffold + CSS | 5 min |
| 2 | Seeded RNG | 2 min |
| 3 | Name generator | 3 min |
| 4 | Race templates (6 silhouettes) | 10 min |
| 5 | Portrait generation | 3 min |
| 6 | Card rendering | 5 min |
| 7 | Selection + screen transitions | 5 min |
| 8 | Waiting, result, narratives | 5 min |
| 9 | Polish + push | 3 min |

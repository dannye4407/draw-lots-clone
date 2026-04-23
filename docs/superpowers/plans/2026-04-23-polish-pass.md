# Polish Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add extended peek, tiered celebrations, synthesized sound, and ambient micro-polish to Chase the Ace without breaking the zero-dependency single-file architecture.

**Architecture:** All game-side changes go inline in `index.html` (existing pattern). Admin config gets an optional `value` field per prize row in `admin.html`. Audio uses the Web Audio API with synthesized oscillators — no asset files. Particle effects use a canvas-based system. All changes stay backward-compatible with existing saved configs.

**Tech Stack:** Vanilla HTML5 / CSS3 / ES6, Web Audio API, Canvas 2D API. No build step, no dependencies, no package manager.

**Spec:** `docs/superpowers/specs/2026-04-22-polish-pass-design.md`

**Test strategy:** This project has no automated test suite. Each task's verification is a concrete browser click-through with expected observations. Verification replaces the "write failing test → run → implement → pass" TDD loop.

---

## Task Order & Dependencies

Linear: 1 → 2 → 3 → 4 → 5 → 6

- **Task 1** adds `value` data to saved configs. Task 3 reads it.
- **Task 2** builds the audio module. Tasks 3, 4, 5 call its `sfx*()` functions.
- **Tasks 4 and 5** together deliver tier classification + all celebration effects.
- **Task 6** is standalone ambient polish.

Each task commits independently and leaves the app playable.

**Shared deck/rank constants used across tasks:**

```js
// Already exists in index.html
const RANKS = ['A','2','3','4','5','6','7','8','9','10','J','Q','K'];

// Helper order (low → high) for comparing prize ranks. Add near top of <script>.
const RANK_ORDER = ['2','3','4','5','6','7','8','9','10','J','Q','K','A'];
```

---

## Task 1: Admin — optional £ value field per prize row

**Files:**
- Modify: `admin.html` — header row, prize row template in `addPrizeRow`, save logic, load logic, CSS width for new input

**Rationale:** Task 3's toast reads `config.prizes[*].value` to render the "Prizes £X–£Y" line. Ship the data model first.

- [ ] **Step 1: Add "£ Value" header to the prize table**

Open `admin.html`. Find the `<thead>` block (around lines 143-151). Add a new `<th>` between `Prize Name` and `Slots`:

Before:
```html
<tr>
  <th>Suit</th>
  <th>Rank</th>
  <th>Prize Name</th>
  <th>Slots</th>
  <th></th>
</tr>
```

After:
```html
<tr>
  <th>Suit</th>
  <th>Rank</th>
  <th>Prize Name</th>
  <th>£ Value</th>
  <th>Slots</th>
  <th></th>
</tr>
```

- [ ] **Step 2: Update empty-state row colspan from 5 to 6**

Find both occurrences of `colspan="5"` in `admin.html` (around lines 154 and 223). Change both to `colspan="6"`.

- [ ] **Step 3: Add value input to prize row template in `addPrizeRow`**

Find the `addPrizeRow(data = {})` function. Modify the `tr.innerHTML` template to add a value input after the prize-name cell. Also add a `value` default to the row fields.

Before:
```js
tr.innerHTML = `
  <td><select class="suit-select" onchange="updateRankOptions(this)">${suitOpts}</select></td>
  <td><select class="rank-select">${rankOpts}</select></td>
  <td><input type="text"   class="prize-name"  placeholder="Prize name" value="${escAttr(data.prize || '')}"></td>
  <td><input type="number" class="prize-slots" min="1" value="${escAttr(data.slots || 1)}" onchange="updateTotal()"></td>
  <td><button class="btn-remove" onclick="removeRow(this)" title="Remove">×</button></td>
`;
```

After:
```js
const valueAttr = (data.value === 0 || data.value) ? escAttr(data.value) : '';
tr.innerHTML = `
  <td><select class="suit-select" onchange="updateRankOptions(this)">${suitOpts}</select></td>
  <td><select class="rank-select">${rankOpts}</select></td>
  <td><input type="text"   class="prize-name"  placeholder="Prize name" value="${escAttr(data.prize || '')}"></td>
  <td><input type="number" class="prize-value" min="0" step="0.01" placeholder="£" value="${valueAttr}"></td>
  <td><input type="number" class="prize-slots" min="1" value="${escAttr(data.slots || 1)}" onchange="updateTotal()"></td>
  <td><button class="btn-remove" onclick="removeRow(this)" title="Remove">×</button></td>
`;
```

- [ ] **Step 4: Add CSS width rule for the new input**

Find the existing `.prize-table input[type="number"] { width: 80px; }` (around line 86). Replace with two targeted rules so value and slots can have different widths:

Before:
```css
.prize-table input[type="number"] { width: 80px; }
```

After:
```css
.prize-table .prize-slots { width: 80px; }
.prize-table .prize-value { width: 90px; }
```

- [ ] **Step 5: Capture value in `saveConfig`**

Find the save loop (around lines 245-256). Add value extraction and include it in the pushed prize object.

Before:
```js
for (const row of rows) {
  const suit  = row.querySelector('.suit-select').value;
  const rank  = row.querySelector('.rank-select').value;
  const prize = row.querySelector('.prize-name').value.trim();
  const slots = parseInt(row.querySelector('.prize-slots').value);
  if (!prize)          { alert('All prize rows must have a prize name.');    return; }
  if (!slots || slots < 1) { alert('All prize rows must have at least 1 slot.'); return; }
  const key = `${suit}-${rank}`;
  if (seen.has(key))   { alert(`Duplicate card: ${suit} ${rank}.`);          return; }
  seen.add(key);
  prizes.push({ suit, rank, prize, slots });
}
```

After:
```js
for (const row of rows) {
  const suit  = row.querySelector('.suit-select').value;
  const rank  = row.querySelector('.rank-select').value;
  const prize = row.querySelector('.prize-name').value.trim();
  const slots = parseInt(row.querySelector('.prize-slots').value);
  const rawValue = row.querySelector('.prize-value').value.trim();
  const value = rawValue === '' ? null : Number(rawValue);
  if (!prize)          { alert('All prize rows must have a prize name.');    return; }
  if (!slots || slots < 1) { alert('All prize rows must have at least 1 slot.'); return; }
  if (rawValue !== '' && (!isFinite(value) || value < 0)) {
    alert('Prize value must be a non-negative number.');
    return;
  }
  const key = `${suit}-${rank}`;
  if (seen.has(key))   { alert(`Duplicate card: ${suit} ${rank}.`);          return; }
  seen.add(key);
  prizes.push({ suit, rank, prize, slots, value });
}
```

- [ ] **Step 6: Make `loadConfig` pass value through to `addPrizeRow`**

`loadConfig` already calls `addPrizeRow(p)`, and `addPrizeRow` now reads `data.value`. No explicit change needed — but the filter line should preserve prizes even if value is missing.

Verify the filter line in `loadConfig` (around line 276) still looks like this (no changes needed, but confirm):

```js
config.prizes = config.prizes.filter(p => p && typeof p === 'object' && p.suit && p.rank);
```

No rejection of missing/null `value` — the default `''` in the row input for missing values is the intended fallback.

- [ ] **Step 7: Verify in browser**

Open `admin.html` in a browser.

1. Click "+ Add Prize" — new row should show a `£` input between prize name and slots.
2. Add a prize: ♠ / A / "Grand Prize" / £500 / 1. Click "Save Configuration". Should show "✓ Saved".
3. Reload the page. The saved prize should load back, value field showing `500`.
4. Add another prize with a blank value. Save. Reload. Blank value should still be blank (not `null` text, not `0`).
5. Try entering `-5` as a value and saving — should alert "Prize value must be a non-negative number."
6. Open browser devtools → Application → Local Storage → `dlConfig`. Confirm shape includes `value` key (number or null) per prize.

- [ ] **Step 8: Commit**

```bash
git add admin.html
git commit -m "feat(admin): optional £ value field per prize row"
git push origin main
```

---

## Task 2: Audio module + mute toggle

**Files:**
- Modify: `index.html` — add audio module in `<script>`, add mute button in header, call `updateMuteIcon` from `init()`

**Rationale:** Subsequent tasks call `sfxPeek()`, `sfxToast()`, `sfxFlip()`, `sfxTier('grand'|'win'|'sad')`. Build once, wire in place.

- [ ] **Step 1: Add mute button to header markup**

Open `index.html`. Find the `.header-actions` div (around line 219).

Before:
```html
<div class="header-actions">
  <a class="btn" href="admin.html">⚙ Admin</a>
  <button class="btn btn-icon" id="theme-btn" onclick="toggleTheme()" title="Toggle theme" aria-label="Toggle light/dark theme">☀️</button>
</div>
```

After:
```html
<div class="header-actions">
  <a class="btn" href="admin.html">⚙ Admin</a>
  <button class="btn btn-icon" id="mute-btn"  onclick="toggleMute()"  title="Toggle sound"  aria-label="Toggle sound">🔊</button>
  <button class="btn btn-icon" id="theme-btn" onclick="toggleTheme()" title="Toggle theme" aria-label="Toggle light/dark theme">☀️</button>
</div>
```

- [ ] **Step 2: Add the audio module inside `<script>`**

Find the `<script>` tag and the existing `let gameActive = false;` line (around line 337). Insert the audio module immediately after it (above the `// ── RNG ──` section).

```js
    // ── Audio ─────────────────────────────────────────────────────────────
    let _audioCtx = null;
    function audio() {
      if (!_audioCtx) {
        const Ctor = window.AudioContext || window.webkitAudioContext;
        if (!Ctor) return null;
        _audioCtx = new Ctor();
      }
      if (_audioCtx.state === 'suspended') _audioCtx.resume();
      return _audioCtx;
    }

    function isMuted() { return localStorage.getItem('dlMute') === '1'; }

    function updateMuteIcon() {
      const btn = document.getElementById('mute-btn');
      if (btn) btn.textContent = isMuted() ? '🔇' : '🔊';
    }

    function toggleMute() {
      localStorage.setItem('dlMute', isMuted() ? '0' : '1');
      updateMuteIcon();
    }

    function _env(gainNode, t0, peak, attack, decay, sustain, release, duration) {
      const g = gainNode.gain;
      g.setValueAtTime(0, t0);
      g.linearRampToValueAtTime(peak, t0 + attack);
      g.linearRampToValueAtTime(peak * sustain, t0 + attack + decay);
      g.setValueAtTime(peak * sustain, t0 + duration - release);
      g.linearRampToValueAtTime(0, t0 + duration);
    }

    function _master(ctx) {
      const g = ctx.createGain();
      g.gain.value = 0.7;
      g.connect(ctx.destination);
      return g;
    }

    function _noiseBuffer(ctx, durationSec) {
      const n = Math.max(1, Math.floor(ctx.sampleRate * durationSec));
      const buf = ctx.createBuffer(1, n, ctx.sampleRate);
      const data = buf.getChannelData(0);
      for (let i = 0; i < n; i++) data[i] = Math.random() * 2 - 1;
      return buf;
    }

    function sfxPeek() {
      if (isMuted()) return;
      const ctx = audio(); if (!ctx) return;
      const t0 = ctx.currentTime;
      const duration = 0.15;
      const master = _master(ctx);
      const src = ctx.createBufferSource();
      src.buffer = _noiseBuffer(ctx, duration);
      const filter = ctx.createBiquadFilter();
      filter.type = 'bandpass'; filter.frequency.value = 2000; filter.Q.value = 5;
      const gain = ctx.createGain();
      _env(gain, t0, 0.25, 0.02, 0, 0, 0.13, duration);
      src.connect(filter).connect(gain).connect(master);
      src.start(t0); src.stop(t0 + duration);
    }

    function sfxToast() {
      if (isMuted()) return;
      const ctx = audio(); if (!ctx) return;
      const t0 = ctx.currentTime;
      const duration = 0.08;
      const master = _master(ctx);
      const osc = ctx.createOscillator();
      osc.type = 'sine'; osc.frequency.value = 900;
      const gain = ctx.createGain();
      _env(gain, t0, 0.18, 0.005, 0.01, 0.7, 0.065, duration);
      osc.connect(gain).connect(master);
      osc.start(t0); osc.stop(t0 + duration);
    }

    function sfxFlip() {
      if (isMuted()) return;
      const ctx = audio(); if (!ctx) return;
      const t0 = ctx.currentTime;
      const duration = 0.2;
      const master = _master(ctx);
      const src = ctx.createBufferSource();
      src.buffer = _noiseBuffer(ctx, duration);
      const filter = ctx.createBiquadFilter();
      filter.type = 'lowpass';
      filter.frequency.setValueAtTime(800, t0);
      filter.frequency.linearRampToValueAtTime(200, t0 + duration);
      const gain = ctx.createGain();
      _env(gain, t0, 0.22, 0.01, 0.05, 0.5, 0.14, duration);
      src.connect(filter).connect(gain).connect(master);
      src.start(t0); src.stop(t0 + duration);
    }

    function _playNote(ctx, master, when, freq, noteDur, peak) {
      const mix = ctx.createGain(); mix.gain.value = 1;
      ['sine', 'triangle'].forEach((type, i) => {
        const osc = ctx.createOscillator();
        osc.type = type; osc.frequency.value = freq;
        const subG = ctx.createGain();
        subG.gain.value = i === 0 ? 1 : 0.5;
        osc.connect(subG).connect(mix);
        osc.start(when); osc.stop(when + noteDur);
      });
      const gain = ctx.createGain();
      _env(gain, when, peak, 0.01, 0.05, 0.4, 0.12, noteDur);
      mix.connect(gain).connect(master);
    }

    function sfxTier(tier) {
      if (isMuted()) return;
      const ctx = audio(); if (!ctx) return;
      const t0 = ctx.currentTime;
      const master = _master(ctx);
      if (tier === 'grand') {
        const notes = [523.25, 659.25, 783.99, 1046.5];
        notes.forEach((f, i) => _playNote(ctx, master, t0 + i * 0.18, f, 0.15, 0.3));
      } else if (tier === 'win') {
        const osc = ctx.createOscillator();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(659.25, t0);
        osc.frequency.setValueAtTime(783.99, t0 + 0.15);
        const gain = ctx.createGain();
        _env(gain, t0, 0.25, 0.01, 0.05, 0.6, 0.12, 0.35);
        osc.connect(gain).connect(master);
        osc.start(t0); osc.stop(t0 + 0.35);
      } else if (tier === 'sad') {
        const osc = ctx.createOscillator();
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(392, t0);
        osc.frequency.linearRampToValueAtTime(220, t0 + 0.7);
        const gain = ctx.createGain();
        _env(gain, t0, 0.2, 0.02, 0.1, 0.3, 0.6, 0.7);
        osc.connect(gain).connect(master);
        osc.start(t0); osc.stop(t0 + 0.7);
      }
    }
```

- [ ] **Step 3: Call `updateMuteIcon()` from `init()`**

Find the `function init()` (around line 544 after prior edits). Add the call next to `applyTheme()`:

Before:
```js
function init() {
  applyTheme();
  const raw = localStorage.getItem('dlConfig');
  ...
}
```

After:
```js
function init() {
  applyTheme();
  updateMuteIcon();
  const raw = localStorage.getItem('dlConfig');
  ...
}
```

- [ ] **Step 4: Wire existing flow to audio**

Find `handleCardClick` (around line 411). Find the current calls that happen at the moments we want to play sounds, and add the three calls:

In the existing `spadeCornerReveal` body (before Task 3 renames it), find `peek.classList.add('active');` and add `sfxPeek();` on the line above. Find `toast.classList.remove('hidden');` and add `sfxToast();` on the line above.

In `handleCardClick`, find `cardEl.classList.add('flipped', 'revealed');` and add `sfxFlip();` on the line immediately above.

In the `startGame()` function (title screen → grid), add `sfxToast();` as the first line. In the `playAgain()` function, add `sfxToast();` as the first line.

Concretely, these five additions are:

```js
// in spadeCornerReveal (Task 3 renames this to cornerPeek)
sfxPeek();
peek.classList.add('active');

// …later, still in spadeCornerReveal…
sfxToast();
toast.classList.remove('hidden');

// in handleCardClick, just before the flip
sfxFlip();
cardEl.classList.add('flipped', 'revealed');

// startGame() — first line of body
sfxToast();

// playAgain() — first line of body
sfxToast();
```

- [ ] **Step 5: Verify in browser**

Open `index.html` with a config present. You should hear sounds:

1. Click "Pick a Card" on title screen → soft blip (toast)
2. Click a SPADE (if one is configured, or use devtools to force one via `dlConfig`): you should hear three sounds — peek slide, toast blip, flip whoosh — in that order
3. Non-spade click: only hear the flip whoosh (extended peek lands in Task 3)
4. Click 🔊 icon in header → changes to 🔇
5. Click a card again → no sound plays
6. Refresh page → 🔇 persists; sound stays muted
7. Click 🔇ck → back to 🔊, sounds play again

Note: if your browser blocks audio until user gesture, the first sound might be silent but subsequent ones work. This is expected.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: Web Audio SFX module with mute toggle"
git push origin main
```

---

## Task 3: Extended Peek — every suit, dynamic toast

**Files:**
- Modify: `index.html` — peek toast HTML (structure for 3 lines), peek CSS (colour classes, line sizing), rename `spadeCornerReveal` → `cornerPeek`, add `buildPeekText` helper, remove spade-only gate in `handleCardClick`

**Rationale:** Every pick gets the suspense beat. Toast content is derived from the live config.

- [ ] **Step 1: Restructure the peek-toast HTML**

Find the `#peek-toast` block in the HTML body (added earlier, around line 251). Replace:

Before:
```html
<div id="peek-toast" class="hidden">
  <span class="toast-suit">♠</span>
  It's a Spade!<br>
  <span class="toast-sub">Let's see the rest of the card…</span>
</div>
```

After:
```html
<div id="peek-toast" class="hidden">
  <span class="toast-suit" id="toast-suit"></span>
  <span class="toast-line1" id="toast-line1"></span>
  <span class="toast-line2" id="toast-line2"></span>
  <span class="toast-line3" id="toast-line3"></span>
</div>
```

- [ ] **Step 2: Update peek-toast CSS to support three lines + suit colour**

Find the existing `#peek-toast` CSS block (added earlier, around line 260-280 region). Replace the existing rules with:

Before:
```css
#peek-toast {
  position: fixed;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(10, 25, 15, 0.92);
  color: #fff;
  border: 1px solid rgba(251,191,36,0.35);
  border-radius: 14px;
  padding: 20px 32px;
  font-size: 1.1rem;
  font-weight: 700;
  text-align: center;
  z-index: 600;
  pointer-events: none;
  line-height: 1.5;
}
#peek-toast.hidden { display: none; }
#peek-toast .toast-suit { font-size: 2rem; display: block; margin-bottom: 6px; }
#peek-toast .toast-sub  { font-size: 0.88rem; font-weight: 400; color: #94a3b8; }
@keyframes toast-pop {
  from { transform: translate(-50%, -50%) scale(0.8); opacity: 0; }
  to   { transform: translate(-50%, -50%) scale(1);   opacity: 1; }
}
#peek-toast:not(.hidden) { animation: toast-pop 0.3s cubic-bezier(0.34, 1.56, 0.64, 1) both; }
```

After:
```css
#peek-toast {
  position: fixed;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(10, 25, 15, 0.92);
  color: #fff;
  border: 1px solid rgba(251,191,36,0.35);
  border-radius: 14px;
  padding: 20px 32px;
  min-width: 280px;
  text-align: center;
  z-index: 600;
  pointer-events: none;
  line-height: 1.4;
}
#peek-toast.hidden { display: none; }
#peek-toast > span { display: block; }
#peek-toast .toast-suit  { font-size: 2rem; margin-bottom: 4px; }
#peek-toast .toast-line1 { font-size: 1.3rem; font-weight: 800; margin-bottom: 6px; }
#peek-toast .toast-line2 { font-size: 1rem;   font-weight: 600; color: #f1f5f9; margin-bottom: 4px; }
#peek-toast .toast-line3 { font-size: 0.85rem; font-weight: 400; color: #94a3b8; }
#peek-toast .toast-line3:empty { display: none; }
#peek-toast.suit-red  .toast-suit { color: #ef4444; }
#peek-toast.suit-black .toast-suit { color: #fff; }
@keyframes toast-pop {
  from { transform: translate(-50%, -50%) scale(0.8); opacity: 0; }
  to   { transform: translate(-50%, -50%) scale(1);   opacity: 1; }
}
#peek-toast:not(.hidden) { animation: toast-pop 0.3s cubic-bezier(0.34, 1.56, 0.64, 1) both; }
```

- [ ] **Step 3: Add suit colour classes to `.card-peek`**

Find the `.card-peek` CSS block. Add these sibling rules below the `@keyframes peek-open` block:

```css
.card-peek.peek-red   { color: #c0392b; }
.card-peek.peek-black { color: #1a202c; }
```

(The default `color` in the existing `.card-peek` rule can stay — it'll be overridden.)

- [ ] **Step 4: Add RANK_ORDER helper near top of script**

Find the `const RANKS = […]` line (around line 325). Add immediately below:

```js
const RANK_ORDER = ['2','3','4','5','6','7','8','9','10','J','Q','K','A'];
```

- [ ] **Step 5: Add `buildPeekText` function**

Insert this function above `spadeCornerReveal` in the script (the section will become the "Corner peek" block):

```js
    // ── Corner peek (extended to all suits) ───────────────────────────────
    function buildPeekText(config, outcome) {
      const SUIT_NAME = { spades: 'Spade', hearts: 'Heart', diamonds: 'Diamond', clubs: 'Club', joker: 'Joker' };
      const RANK_LABEL = { A: 'Ace', J: 'Jack', Q: 'Queen', K: 'King' };

      const prizesInSuit = config.prizes.filter(p => p.suit === outcome.suit);
      const line1 = `A ${SUIT_NAME[outcome.suit] || outcome.suit}!`;

      let line2;
      if (prizesInSuit.length === 0) {
        line2 = 'No winners in this suit…';
      } else if (outcome.suit === 'joker') {
        const hasRJ = prizesInSuit.some(p => p.rank === 'RJ');
        const hasBJ = prizesInSuit.some(p => p.rank === 'BJ');
        if (hasRJ && hasBJ) line2 = 'Red or Black Joker wins!';
        else if (hasRJ)     line2 = 'Red Joker wins!';
        else                line2 = 'Black Joker wins!';
      } else {
        const winningRanks = [...new Set(prizesInSuit.map(p => p.rank))]
          .sort((a, b) => RANK_ORDER.indexOf(a) - RANK_ORDER.indexOf(b));

        if (winningRanks.length === 13) {
          line2 = `All ${SUIT_NAME[outcome.suit]}s are winners!`;
          const byRank = [...prizesInSuit].sort((a, b) =>
            RANK_ORDER.indexOf(a.rank) - RANK_ORDER.indexOf(b.rank));
          const allHaveValues = byRank.every(p => typeof p.value === 'number');
          const monotonic = allHaveValues && byRank.every((p, i) =>
            i === 0 || p.value >= byRank[i - 1].value);
          if (monotonic) line2 += ' The higher the card, the bigger the prize!';
        } else {
          const labels = winningRanks.map(r => RANK_LABEL[r] || r);
          let list;
          if      (labels.length === 1) list = labels[0];
          else if (labels.length === 2) list = `${labels[0]} or ${labels[1]}`;
          else list = labels.slice(0, -1).join(', ') + ', or ' + labels[labels.length - 1];
          line2 = `Any ${list} wins!`;
        }
      }

      let line3 = '';
      const vals = prizesInSuit.map(p => p.value).filter(v => typeof v === 'number');
      if (vals.length > 0) {
        const fmt = v => Number.isInteger(v) ? `£${v}` : `£${v.toFixed(2)}`;
        const uniq = [...new Set(vals)];
        if (uniq.length === 1) line3 = `Prize ${fmt(uniq[0])}`;
        else line3 = `Prizes ${fmt(Math.min(...vals))}–${fmt(Math.max(...vals))}`;
      }

      return { line1, line2, line3 };
    }
```

- [ ] **Step 6: Rewrite `spadeCornerReveal` as `cornerPeek`**

Find the existing `function spadeCornerReveal(cardEl, outcome) { … }` (around line 515). Replace entirely with:

```js
    function cornerPeek(cardEl, outcome, config) {
      return new Promise(resolve => {
        const peek = cardEl.querySelector('.card-peek');
        const sym  = peek.querySelector('.peek-sym');
        sym.textContent = outcome.symbol;
        peek.classList.remove('peek-red', 'peek-black');
        if (outcome.suit === 'hearts' || outcome.suit === 'diamonds') peek.classList.add('peek-red');
        else if (outcome.suit === 'spades' || outcome.suit === 'clubs') peek.classList.add('peek-black');
        sfxPeek();
        peek.classList.add('active');

        const text = buildPeekText(config, outcome);
        const toast = document.getElementById('peek-toast');
        document.getElementById('toast-suit').textContent = outcome.suit === 'joker' ? '🃏' : outcome.symbol;
        document.getElementById('toast-line1').textContent = text.line1;
        document.getElementById('toast-line2').textContent = text.line2;
        document.getElementById('toast-line3').textContent = text.line3;
        toast.classList.remove('suit-red', 'suit-black');
        if (outcome.suit === 'hearts' || outcome.suit === 'diamonds') toast.classList.add('suit-red');
        else if (outcome.suit === 'spades' || outcome.suit === 'clubs') toast.classList.add('suit-black');

        setTimeout(() => {
          sfxToast();
          toast.classList.remove('hidden');
          setTimeout(() => {
            toast.classList.add('hidden');
            resolve();
          }, 2500);
        }, 850);
      });
    }
```

- [ ] **Step 7: Remove spade-only gate in `handleCardClick`**

Find the block in `handleCardClick` (around line 425):

Before:
```js
if (outcome.suit === 'spades') {
  await spadeCornerReveal(cardEl, outcome);
}
```

After:
```js
await cornerPeek(cardEl, outcome, config);
```

- [ ] **Step 8: Verify in browser**

Open `index.html` with a config that includes prize cards in multiple suits, ideally a mix with some suits that have no prizes at all, and some with values.

1. Click a ♥ card → corner peels open showing ♥ in RED colour → toast appears centred, shows `A Heart!` with red emoji, then `Any ...` or `All Hearts are winners!` per config, then value range line if configured → after 2.5s toast dismisses → card flips → modal opens
2. Click a ♠ card → same flow with `A Spade!` and black emoji
3. Click a ♦ card → red colour
4. Click a ♣ card → black colour
5. If any suit has no prizes in config → clicking one shows `No winners in this suit…`
6. If all 13 of a suit are prizes with increasing values → toast says `All Xs are winners! The higher the card, the bigger the prize!`
7. Joker pick → shows `A Joker!` with 🃏 emoji and appropriate Red/Black Joker message
8. Mute is still respected (no `sfxPeek`/`sfxToast` sounds when muted)

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: extended peek with dynamic toast for every suit"
git push origin main
```

---

## Task 4: Tier classification + Sad dim + Modal variants

**Files:**
- Modify: `index.html` — add `getTier`, add modal variant CSS classes, add `fxDim` helper, update `showModal` to classify + apply + play tier sound

**Rationale:** Task 4 wires the tiered response for the "sad" case (no prize) and the per-tier modal pop feel. Task 5 adds the canvas particle effects layered on top.

- [ ] **Step 1: Add modal variant classes + dim overlay CSS**

Find the existing `.modal-box` animation rule (around line 198, the `animation: pop-in ...` line). After the `@keyframes pop-in` block, add these rules:

```css
    /* Modal variants per celebration tier */
    .modal-box.modal-grand {
      animation: pop-in-grand 0.5s cubic-bezier(0.34, 1.8, 0.4, 1) both;
      box-shadow: 0 24px 64px rgba(0,0,0,0.45), 0 0 80px rgba(251, 191, 36, 0.55);
    }
    .modal-box.modal-sad {
      animation: pop-in-sad 0.4s cubic-bezier(0.4, 0, 0.4, 1) both;
    }
    @keyframes pop-in-grand {
      0%   { transform: scale(0.6); opacity: 0; }
      60%  { transform: scale(1.08); opacity: 1; }
      100% { transform: scale(1);   opacity: 1; }
    }
    @keyframes pop-in-sad {
      from { transform: scale(0.92); opacity: 0; }
      to   { transform: scale(1);    opacity: 1; }
    }

    /* Full-viewport dim overlay (sad tier) */
    .fx-dim {
      position: fixed;
      inset: 0;
      background: rgba(0, 0, 0, 0.2);
      z-index: 199;
      pointer-events: none;
      animation: fx-dim-pulse 0.3s ease-out both;
    }
    @keyframes fx-dim-pulse {
      0%   { opacity: 0; }
      50%  { opacity: 1; }
      100% { opacity: 0; }
    }
```

- [ ] **Step 2: Add `getTier` + `fxDim` helpers**

Find the `// ── Modal ──` comment block (around line 460). Add these helpers above the `function showModal` line:

```js
    function getTier(outcome, config) {
      if (outcome.prize === null) return 'sad';
      const match = config.prizes.find(p => p.suit === outcome.suit && p.rank === outcome.rank);
      if (!match) return 'sad';
      const minSlots = Math.min(...config.prizes.map(p => p.slots));
      return match.slots === minSlots ? 'grand' : 'win';
    }

    function fxDim() {
      const d = document.createElement('div');
      d.className = 'fx-dim';
      document.body.appendChild(d);
      setTimeout(() => d.remove(), 300);
    }
```

- [ ] **Step 3: Update `showModal` signature to accept config, and classify tier**

The existing `showModal(outcome)` contains an `innerHTML` assignment to `#modal-emoji` that holds a very long base64 PNG string (Sarim image) in the no-prize branch. **Do not rewrite or touch that line.** This step is a minimum-diff patch:

1. **Change the function signature** from `function showModal(outcome) {` to `function showModal(outcome, config) {`.

2. **Insert these 6 new lines at the top of the function body** (immediately after the existing `const isWin = !!outcome.prize;` line):

```js
const tier = getTier(outcome, config);

const box = document.querySelector('#prize-modal .modal-box');
box.classList.remove('modal-grand', 'modal-sad');
if (tier === 'grand') box.classList.add('modal-grand');
else if (tier === 'sad') box.classList.add('modal-sad');
```

3. **Insert these 2 new lines** immediately before the existing final `document.getElementById('prize-modal').classList.remove('hidden');` line:

```js
if (tier === 'sad') fxDim();
sfxTier(tier);
```

4. **Leave all other lines unchanged**, including the Sarim base64 innerHTML assignment.

- [ ] **Step 4: Update `showModal` caller in `handleCardClick`**

Find the line in `handleCardClick` (inside the `transitionend` listener) that calls `showModal(outcome)`. `config` is already in scope as a local variable of `handleCardClick`. Pass it through:

Before:
```js
cardEl.querySelector('.card-inner').addEventListener('transitionend', () => {
  showModal(outcome);
}, { once: true });
```

After:
```js
cardEl.querySelector('.card-inner').addEventListener('transitionend', () => {
  showModal(outcome, config);
}, { once: true });
```

- [ ] **Step 5: Verify in browser**

Open `index.html` with a config where:
- ♠ A has the smallest slot count (e.g., 1) — this will be the grand tier
- Other prize cards have larger slot counts (e.g., 10, 50)
- Keep some non-prize cards available too

1. Force a grand-tier outcome (easiest: temporarily set pool size very small and make A♠ the only prize; click any card). You should hear the arpeggio + see a slightly bigger bouncier pop on the modal with a gold glow. **No confetti yet — Task 5.**
2. Force a win-tier outcome (a prize card with higher slot count than A♠). You should hear the two-note up-chime. Modal pop is standard.
3. Force a no-prize outcome (click until one lands). You should see a brief screen-wide dim pulse and hear the descending sawtooth. Modal pop is slower and less bouncy.
4. Mute still respected on all three tiers (no tier sound when muted). The dim/glow remains — only the audio is muted.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: tier classification, sad dim, and modal tier variants"
git push origin main
```

---

## Task 5: Canvas FX — grand confetti + win sparkles + flash + shake

**Files:**
- Modify: `index.html` — add canvas FX module (`createFxCanvas`, `runConfetti`, `runSparkles`, `fxFlash`, `fxShake`), add shake keyframes + flash styles, wire into `showModal`

- [ ] **Step 1: Add FX canvas + flash + shake CSS**

Find the `@keyframes fx-dim-pulse` block added in Task 4. Immediately after it, add:

```css
    /* Full-screen flash (grand tier) */
    .fx-flash {
      position: fixed;
      inset: 0;
      background: rgba(251, 191, 36, 0.35);
      z-index: 650;
      pointer-events: none;
      animation: fx-flash-pulse 0.5s ease-out both;
    }
    @keyframes fx-flash-pulse {
      0%   { opacity: 0; }
      35%  { opacity: 1; }
      100% { opacity: 0; }
    }

    /* Grand-tier body shake */
    body.shake-grand { animation: shake-grand 0.4s ease-in-out both; }
    @keyframes shake-grand {
      0%, 100% { transform: translateX(0); }
      15% { transform: translateX(-6px); }
      30% { transform: translateX(5px); }
      45% { transform: translateX(-4px); }
      60% { transform: translateX(3px); }
      75% { transform: translateX(-2px); }
      90% { transform: translateX(1px); }
    }

    /* Shared canvas overlay for particle effects */
    .fx-canvas { position: fixed; inset: 0; pointer-events: none; z-index: 700; }
```

- [ ] **Step 2: Add canvas FX module to script**

Find the `function getTier(...)` added in Task 4. Insert the canvas FX module immediately above it (in a new `// ── Canvas FX ──` section):

```js
    // ── Canvas FX ─────────────────────────────────────────────────────────
    function createFxCanvas() {
      const c = document.createElement('canvas');
      c.className = 'fx-canvas';
      c.width = window.innerWidth;
      c.height = window.innerHeight;
      document.body.appendChild(c);
      return c;
    }

    function runConfetti() {
      const c = createFxCanvas();
      const ctx = c.getContext('2d');
      const colors = ['#fbbf24', '#f59e0b', '#d97706', '#b91c1c', '#ffffff'];
      const particles = [];
      for (let i = 0; i < 150; i++) {
        particles.push({
          x: Math.random() * c.width,
          y: -10 - Math.random() * 100,
          vx: (Math.random() - 0.5) * 200,
          vy: 100 + Math.random() * 200,
          angle: Math.random() * Math.PI * 2,
          spin: (Math.random() - 0.5) * 6,
          w: 4, h: 8,
          color: colors[Math.floor(Math.random() * colors.length)],
          dead: false,
        });
      }
      const gravity = 400;
      const start = performance.now();
      let last = start;
      function frame(now) {
        const dt = Math.min(0.05, (now - last) / 1000); last = now;
        ctx.clearRect(0, 0, c.width, c.height);
        let alive = 0;
        for (const p of particles) {
          if (p.dead) continue;
          p.vy += gravity * dt;
          p.x += p.vx * dt;
          p.y += p.vy * dt;
          p.angle += p.spin * dt;
          if (p.y > c.height + 50) { p.dead = true; continue; }
          ctx.save();
          ctx.translate(p.x, p.y);
          ctx.rotate(p.angle);
          ctx.fillStyle = p.color;
          ctx.fillRect(-p.w / 2, -p.h / 2, p.w, p.h);
          ctx.restore();
          alive++;
        }
        if (alive > 0 && now - start < 4000) requestAnimationFrame(frame);
        else c.remove();
      }
      requestAnimationFrame(frame);
    }

    function runSparkles(cx, cy) {
      const c = createFxCanvas();
      const ctx = c.getContext('2d');
      const particles = [];
      for (let i = 0; i < 30; i++) {
        const angle = Math.random() * Math.PI * 2;
        const speed = 80 + Math.random() * 80;
        particles.push({
          x: cx, y: cy,
          vx: Math.cos(angle) * speed,
          vy: Math.sin(angle) * speed,
          age: 0, life: 1.2, r: 3,
        });
      }
      const gravity = 150;
      let last = performance.now();
      function frame(now) {
        const dt = Math.min(0.05, (now - last) / 1000); last = now;
        ctx.clearRect(0, 0, c.width, c.height);
        let alive = 0;
        for (const p of particles) {
          p.age += dt;
          if (p.age >= p.life) continue;
          p.vy += gravity * dt;
          p.x += p.vx * dt;
          p.y += p.vy * dt;
          const alpha = 1 - (p.age / p.life);
          const radius = p.r * 3;
          const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, radius);
          grad.addColorStop(0, `rgba(255, 215, 0, ${alpha})`);
          grad.addColorStop(1, 'rgba(255, 215, 0, 0)');
          ctx.fillStyle = grad;
          ctx.beginPath();
          ctx.arc(p.x, p.y, radius, 0, Math.PI * 2);
          ctx.fill();
          alive++;
        }
        if (alive > 0) requestAnimationFrame(frame);
        else c.remove();
      }
      requestAnimationFrame(frame);
    }

    function fxFlash() {
      const d = document.createElement('div');
      d.className = 'fx-flash';
      document.body.appendChild(d);
      setTimeout(() => d.remove(), 500);
    }

    function fxShake() {
      document.body.classList.add('shake-grand');
      setTimeout(() => document.body.classList.remove('shake-grand'), 400);
    }
```

- [ ] **Step 3: Wire tier effects into `showModal`**

In `showModal`, after the existing `document.getElementById('prize-modal').classList.remove('hidden');` line, add tier-specific effect triggers.

Before:
```js
if (tier === 'sad') fxDim();
sfxTier(tier);
document.getElementById('prize-modal').classList.remove('hidden');
```

After:
```js
if (tier === 'sad') fxDim();
sfxTier(tier);
document.getElementById('prize-modal').classList.remove('hidden');

if (tier === 'grand') {
  fxFlash();
  fxShake();
  runConfetti();
} else if (tier === 'win') {
  const box = document.querySelector('#prize-modal .modal-box');
  const rect = box.getBoundingClientRect();
  runSparkles(rect.left + rect.width / 2, rect.top + rect.height / 2);
}
```

- [ ] **Step 4: Verify in browser**

1. Grand-tier outcome → screen flashes gold briefly + body shakes for 0.4s + confetti canvas overlays everything with 150 falling particles in gold/crimson/white for ~3s. Hear the arpeggio.
2. Win-tier outcome → gold sparkle burst from the modal's centre, ~1.5s. Hear the up-chime. No flash, no shake, no confetti.
3. Sad-tier outcome → dim pulse only (no particles), descending trombone sound.
4. Multiple rapid clicks (play again → play again → play again on a win tier) don't stack canvases permanently — each canvas element removes itself when its particles expire. Open devtools → Elements and watch `canvas.fx-canvas` elements appear and disappear.
5. Resize browser mid-confetti: particles continue from the snapshot size; acceptable (brief, ends quickly). Do not add a resize listener.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: tiered celebration effects - confetti, sparkles, flash, shake"
git push origin main
```

---

## Task 6: Micro-polish — hover lift, shuffle-in, felt sparkles, title glint

**Files:**
- Modify: `index.html` — hover CSS for `.card`, shuffle-in CSS + renderGrid integration, ambient felt sparkles on init, title glint pseudo-elements

- [ ] **Step 1: Hover lift CSS**

Find the `.card { … }` rule block (around line 109). Add these new rules immediately after the existing `.card.revealed { cursor: default; }` line:

```css
    .card:not(.used):not(.revealed) {
      transition: transform 150ms ease, filter 150ms ease;
      transform-origin: center bottom;
    }
    .card:not(.used):not(.revealed):hover {
      transform: translateY(-5px) rotateX(-4deg);
      filter: drop-shadow(0 6px 10px rgba(0,0,0,0.4));
    }
```

- [ ] **Step 2: Shuffle-in CSS**

In the same CSS section, after the hover rules, add:

```css
    @keyframes deal-in {
      from { opacity: 0; transform: translateY(-30px) rotate(-6deg) scale(0.8); }
      to   { opacity: 1; transform: none; }
    }
    .card.deal-in {
      animation: deal-in 0.5s cubic-bezier(0.2, 1, 0.3, 1) both;
      animation-delay: calc(var(--i, 0) * 15ms);
    }
```

- [ ] **Step 3: Wire shuffle-in into `renderGrid`**

Find `renderGrid()`. Inside the `for` loop, after `card.className = 'card';`, add the two new lines shown:

Before:
```js
for (let i = 0; i < 54; i++) {
  const card = document.createElement('div');
  card.className = 'card';
  card.innerHTML = `
    <div class="card-inner">
      <div class="card-back"></div>
      <div class="card-front"></div>
    </div>
    <div class="card-peek"><span class="peek-sym"></span></div>`;
  card.addEventListener('click', () => handleCardClick(card));
  grid.appendChild(card);
}
```

After:
```js
for (let i = 0; i < 54; i++) {
  const card = document.createElement('div');
  card.className = 'card deal-in';
  card.style.setProperty('--i', i);
  card.innerHTML = `
    <div class="card-inner">
      <div class="card-back"></div>
      <div class="card-front"></div>
    </div>
    <div class="card-peek"><span class="peek-sym"></span></div>`;
  card.addEventListener('click', () => handleCardClick(card));
  grid.appendChild(card);
}
```

- [ ] **Step 4: Felt sparkles CSS + init creation**

In the CSS section, add below the shuffle-in rules:

```css
    .felt-sparkle {
      position: fixed;
      pointer-events: none;
      width: 3px; height: 3px;
      border-radius: 50%;
      background: radial-gradient(circle, rgba(255,255,255,0.85) 0%, rgba(255,255,255,0) 70%);
      opacity: 0;
      animation: sparkle-breathe 6s ease-in-out infinite;
      z-index: 0;
    }
    @keyframes sparkle-breathe {
      0%, 100% { opacity: 0;   transform: scale(0.6); }
      50%      { opacity: 0.7; transform: scale(1.3); }
    }
```

In `init()`, add felt sparkle creation after `updateMuteIcon()`:

Before:
```js
function init() {
  applyTheme();
  updateMuteIcon();
  const raw = localStorage.getItem('dlConfig');
  ...
}
```

After:
```js
function init() {
  applyTheme();
  updateMuteIcon();
  spawnFeltSparkles();
  const raw = localStorage.getItem('dlConfig');
  ...
}

function spawnFeltSparkles() {
  for (let i = 0; i < 8; i++) {
    const s = document.createElement('div');
    s.className = 'felt-sparkle';
    s.style.top  = `${10 + Math.random() * 80}%`;
    s.style.left = `${Math.random() * 100}%`;
    s.style.animationDelay = `${Math.random() * 6}s`;
    document.body.appendChild(s);
  }
}
```

- [ ] **Step 5: Title glint CSS**

Add at the end of the `<style>` block (immediately before `</style>`):

```css
    /* Title glint sweep */
    header h1, .ts-title { position: relative; overflow: hidden; }
    header h1::before, .ts-title::before {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(110deg,
        transparent 30%,
        rgba(255, 255, 255, 0.55) 50%,
        transparent 70%);
      mix-blend-mode: screen;
      transform: translateX(-100%);
      animation: title-glint 5s ease-in-out infinite;
      pointer-events: none;
    }
    @keyframes title-glint {
      0%, 85%   { transform: translateX(-100%); }
      95%, 100% { transform: translateX(200%); }
    }
    .ts-title::before { animation-delay: 2s; }
```

- [ ] **Step 6: Verify in browser**

1. Hover over an unrevealed card → it lifts ~5px with a subtle tilt and shadow. No lift on already-flipped or dimmed cards.
2. On page load → cards "deal in" from above with a staggered animation over ~1.3s. Click "Play Again" after a prize modal → cards re-deal the same way.
3. Look at the green felt background → see 8 subtle white sparkle dots slowly breathing in/out at different timings. They should be very faint — if they're too distracting, their `opacity: 0.7` peak is too high and should be lowered.
4. Watch the header "Chase the Ace ♠" title → a thin bright diagonal shimmer sweeps across every 5s. Same happens to the title-screen title, offset by ~2s.
5. Browser devtools → no console errors. Page still responsive.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: micro-polish - hover lift, shuffle deal-in, felt sparkles, title glint"
git push origin main
```

---

## Self-Review Checklist (for the plan author)

- ✅ **Spec coverage:** Every spec section is implemented:
  - Feature 1 (Extended Peek) → Task 3 + Task 1 (value data) + Task 2 (sounds)
  - Feature 2 (Tiered Celebrations) → Tasks 4 + 5
  - Feature 3 (Sound) → Task 2 + hooks in 3, 4, 5
  - Feature 4 (Micro-polish) → Task 6
- ✅ **Placeholder scan:** no TODOs/TBDs/vague "handle edge cases"/unstated code
- ✅ **Type consistency:** `config.prizes[*].value` is used identically across Tasks 1 (save/load), 3 (buildPeekText), and all comparisons treat `number | null`. Tier strings `'grand' | 'win' | 'sad'` used identically in `getTier`, `sfxTier`, `showModal`.
- ✅ **Single source of truth** for RANK_ORDER (defined once, reused)

## Known limitations (deferred)

- No resize listener on particle canvases — if the user resizes during an effect, the canvas stays at snapshot size. Acceptable because effects last ≤4s.
- Audio context unlock on iOS Safari may take one user gesture to warm up; the first sound may be silent.
- Hover lift is ignored on touch devices (no `:hover` state) — acceptable.
- If the user changes the browser tab mid-pick, `requestAnimationFrame` pauses; effects appear paused too — acceptable.

## Out of scope (per spec)

Session stats, streaks, real audio samples, mobile haptics, admin auth, server-side RNG, daily spin, social sharing, background music loop, non-GBP currency.

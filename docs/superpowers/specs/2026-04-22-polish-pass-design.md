# Polish Pass — Chase the Ace

_Date: 2026-04-22_

## Overview

A four-feature polish pass on the Chase the Ace lottery reveal game. Adds dramatic per-pick suspense, tier-scaled celebrations, synthesized audio, and ambient polish. All changes stay within the zero-dependency vanilla HTML/CSS/JS architecture — no libraries, no assets, no network calls.

**Goals**
- Make every pick feel suspenseful, not just rare spade draws
- Make winning distinctly different from losing; grand wins feel earth-shattering
- Add audio that works offline, with zero licensing / asset shipping
- Polish the idle state so the felt table feels alive

**Non-goals (this pass)**
- Session stats, running totals, streaks
- Server-side RNG
- Admin authentication
- Real audio samples (synthesized only)
- Ambient background music

## Files modified

| File | Changes |
|---|---|
| `index.html` | All game-side polish: peek logic, tiered celebrations with canvas particle FX, audio module, hover/shuffle/sparkle/glint CSS, mute toggle |
| `admin.html` | Optional `£ value` input per prize row; extend save/load to persist it |

localStorage keys:
- `dlConfig` — extended with optional `value` on each prize (backward-compatible)
- `dlMute` — new, `'1'` or `'0'`

---

## Feature 1: Extended Peek

### Flow per pick

1. Player clicks card
2. RNG resolves outcome (unchanged logic)
3. **Corner peel** — 0.8s CSS animation; peek shows the outcome's suit symbol in suit-appropriate colour
   - ♠♣ dark, ♥♦ red, 🃏 emoji (already coloured)
4. **Toast** — centre-screen, 2.5s total (0.2s fade-in, 2.1s visible, 0.2s fade-out)
5. Toast dismisses → card flips → prize modal opens

Total pre-modal time per pick: ~3.8s.

### Toast content generator

Given the outcome's suit, compute from the current `dlConfig`:

```
prizesInSuit   = config.prizes.filter(p => p.suit === outcome.suit)
ranksThatWin   = unique ranks in prizesInSuit, sorted by card order
valuesPresent  = any p.value is a non-null number in prizesInSuit
values         = prizesInSuit.map(p => p.value).filter(v => v != null)
```

Card rank order (low → high): `2, 3, 4, 5, 6, 7, 8, 9, 10, J, Q, K, A`.

**Line 1 (hero):** `"A [Suit]!"` — large font, suit-coloured. For jokers: `"A Joker!"`.

**Line 2 (sub-copy):**

- `prizesInSuit.length === 0` → `"No winners in this suit…"`
- `ranksThatWin.length === 13` (non-joker, all ranks present) → `"All [Suit]s are winners!"`
  - Append `" The higher the card, the bigger the prize!"` **if** values are present **and** monotonically non-decreasing by rank order
- Joker suit: `"Red or Black Joker wins!"` if both configured, else `"[Red|Black] Joker wins!"`
- Otherwise → `"Any [rank list] wins!"`

Natural rank list formatting (A=Ace, J=Jack, Q=Queen, K=King, others as-is):
- 1: `"Jack"`
- 2: `"King or Ace"`
- 3+: `"Jack, Queen, King, or Ace"` (commas + Oxford)

**Line 3 (value range, optional, small):**

- Skip entirely if `!valuesPresent`
- Single distinct value → `"Prize £[value]"`
- Multiple distinct values → `"Prizes £[min]–£[max]"`
- Values formatted with no decimals if integer, two decimals otherwise

### Examples

Sample config (all prize values in GBP):

```
A♠ = £500, K♠ = £100, Q♠ = £50, J♠ = £25, 10♠ = £10, 9♠ = £5, … 2♠ = £1  (all 13 spades)
A♥/A♦/A♣ = £50, K♥/K♦/K♣ = £25, Q♥/Q♦/Q♣ = £15, J♥/J♦/J♣ = £10           (J+ in red/black non-spade)
RJ + BJ = £20
```

Heart pick:
- `A Heart!`
- `Any Jack, Queen, King, or Ace wins!`
- `Prizes £10–£50`

Spade pick:
- `A Spade!`
- `All Spades are winners! The higher the card, the bigger the prize!`
- `Prizes £1–£500`

Joker pick:
- `A Joker!`
- `Red or Black Joker wins!`
- `Prize £20`

### Admin change

Add optional numeric `value` input to each prize row in `admin.html`:

- Input type `number`, `min="0"`, `step="0.01"`, placeholder `"£ value (optional)"`
- Saved to config as `Number(input.value) || null`
- `loadConfig` treats missing / NaN as null
- Blank input is self-explanatory; no extra UI chrome

Config shape becomes: `{ suit, rank, prize, slots, value: number | null }`. Existing saved configs without the field load cleanly (value becomes null for all rows).

---

## Feature 2: Tiered Celebrations

### Tier classification

```
minSlots = min(config.prizes.map(p => p.slots))

tier =
  outcome.prize === null   → 'sad'
  outcome slots === minSlots   → 'grand'
  else                         → 'win'
```

`outcome slots` is the slot count of the matched prize row (matched on both suit AND rank). Tied rarest prizes all count as grand.

### Effects

| Tier | Effects when modal opens |
|---|---|
| Grand | Full-screen confetti (150 particles, 3s), 0.5s gold screen flash, 0.4s body screen-shake, modal pop with amplified scale + gold glow |
| Win | 30-particle gold sparkle burst from modal centre, ~1.5s, standard modal pop |
| Sad | 0.3s screen dim pulse, slower less-bouncy modal entrance, no particles |

### Canvas particle system

Single reusable implementation for both confetti and sparkle:

```
createFxCanvas():
  <canvas class="fx-canvas"> sized to viewport, appended to body
  pointer-events: none, z-index: 700
  returns ctx + particles array

runParticles(ctx, particles):
  requestAnimationFrame loop
  each frame: update positions (dx = vx*dt, dy = vy*dt), apply gravity, rotation
  clear + redraw particles
  remove particles when offscreen or life expired
  when array empty or hard cap (4s) hit, remove canvas
```

No resize listener; effects are short-lived and size is snapshotted at creation.

**Confetti particles** (grand):
- Spawn: 150 over first 0.3s, random x across top, y = -10
- Shape: 4×8px coloured rectangle
- Palette: `#fbbf24` `#f59e0b` `#d97706` `#b91c1c` `#ffffff`
- Velocity: vx ∈ [-100, 100] px/s, vy ∈ [100, 300] px/s
- Gravity: 400 px/s²
- Rotation: random initial angle, angular velocity ∈ [-3, 3] rad/s
- Life: until offscreen-bottom

**Sparkle particles** (win):
- Spawn: 30 at modal centre (computed from `.modal-box` bounding rect)
- Shape: 3px radius, drawn as radial-gradient glow (gold core, transparent edge) via canvas
- Velocity: radial outward, random magnitude 80–160 px/s
- Gravity: 150 px/s²
- Life: 1.2s with linear alpha fade

### Screen flash (grand only)

Full-viewport `<div class="fx-flash">` — `position: fixed; inset: 0; z-index: 650; pointer-events: none; background: rgba(251, 191, 36, 0.35);`. CSS animation 0 → 0.35 → 0 over 0.5s, then element removed.

### Screen shake (grand only)

Body `animation: shake-grand 0.4s`. Keyframes translate between ±6px on X at ~30ms jitter intervals. Applied to `<body>` not the modal, so the whole scene shakes.

### Screen dim (sad only)

Full-viewport `<div class="fx-dim">` — `background: rgba(0, 0, 0, 0.2); z-index: 199;` (below modal's 200). Animates opacity 0 → 0.2 → 0 over 0.3s, then removed.

### Modal pop variants

Existing `.modal-box` pop-in uses `cubic-bezier(0.34, 1.56, 0.64, 1)`. Add modifier classes:
- `.modal-grand` — scale-in from 0.6 → 1 with gold box-shadow glow
- `.modal-sad` — `cubic-bezier(0.4, 0, 0.4, 1)`, 0.4s, no bounce

Applied by `showModal()` based on tier before class flipping.

---

## Feature 3: Synthesized Sound

### AudioContext

- Module-level lazy-init: first call to any `sfx*()` creates the `AudioContext`
- Browser autoplay policy: because first init is inside a click handler, it starts unblocked
- Construction: `new (window.AudioContext || window.webkitAudioContext)()`

### Envelope helper

```
envelope(gainNode, when, { attack, decay, sustain, release, peak, duration })
  - gain.setValueAtTime(0, when)
  - linearRampToValueAtTime(peak, when + attack)
  - linearRampToValueAtTime(peak * sustain, when + attack + decay)
  - setValueAtTime(peak * sustain, when + duration - release)
  - linearRampToValueAtTime(0, when + duration)
```

### Sound library

| Function | Synthesis |
|---|---|
| `sfxPeek()` | 0.15s white-noise BufferSource → BiquadFilter band-pass @ 2000Hz, Q=5 → envelope peak 0.25, a=0.02/d=0/s=0/r=0.13 |
| `sfxToast()` | Sine osc @ 900Hz, 80ms, envelope peak 0.18, a=0.005/d=0.01/s=0.7/r=0.065 |
| `sfxFlip()` | 0.2s noise BufferSource → BiquadFilter low-pass, frequency.linearRampToValueAtTime 800→200Hz, envelope peak 0.22, a=0.01/d=0.05/s=0.5/r=0.14 |
| `sfxTier('grand')` | Arpeggio C5(523.25)→E5(659.25)→G5(783.99)→C6(1046.5), each note 0.15s with 0.18s spacing (total 0.84s). Each note = sine osc + triangle osc half-gain blend, envelope a=0.01/d=0.05/s=0.4/r=0.12, peak 0.3 |
| `sfxTier('win')` | E5 for 0.15s → G5 for 0.2s, sine, envelope peak 0.25, a=0.01/d=0.05/s=0.6/r=0.12 |
| `sfxTier('sad')` | Sawtooth osc starting @ G4 (392), linearRampToValueAtTime to 220Hz over 0.7s. Envelope peak 0.2, a=0.02/d=0.1/s=0.3/r=0.6 |

All connected through a single master GainNode at 0.7 for volume headroom, then to `destination`.

### Hook points

| Event | SFX |
|---|---|
| Card click (start of peek) | `sfxPeek()` |
| Toast appears | `sfxToast()` |
| Card flip starts | `sfxFlip()` |
| Modal opens | `sfxTier(tier)` matching prize tier |
| Title-screen "Pick a Card" button click | `sfxToast()` as soft UI tap |
| Modal "Play Again" click | `sfxToast()` |

### Mute toggle

- New `<button id="mute-btn" class="btn btn-icon">` in header `.header-actions`, immediately before the theme button
- Icon text: `🔊` unmuted, `🔇` muted
- `aria-label="Toggle sound"`, `title="Toggle sound"`
- Persisted in `localStorage['dlMute']` as `'1'` / `'0'`; default `'0'` (unmuted)
- Every SFX function wraps in `if (isMuted()) return;`
- Toggling updates the icon and storage; does not retroactively mute ongoing notes

---

## Feature 4: Micro-polish

### Hover lift

```css
.card:not(.used):not(.revealed) {
  transition: transform 150ms ease, filter 150ms ease;
}
.card:not(.used):not(.revealed):hover {
  transform: translateY(-5px) rotateX(-4deg);
  filter: drop-shadow(0 6px 10px rgba(0, 0, 0, 0.4));
}
```

The card already has `perspective: 700px`, so rotateX lands naturally. `transform-origin: center bottom` added to `.card` so the lift pivots from the base.

### Shuffle-in

In `renderGrid()`, after creating each card:

```js
card.style.setProperty('--i', i);
card.classList.add('deal-in');
```

CSS:
```css
@keyframes deal-in {
  from { opacity: 0; transform: translateY(-30px) rotate(-6deg) scale(0.8); }
  to   { opacity: 1; transform: none; }
}
.card.deal-in {
  animation: deal-in 0.5s cubic-bezier(0.2, 1, 0.3, 1) both;
  animation-delay: calc(var(--i) * 15ms);
}
```

- Total: last card (i=53) finishes at 54×15ms + 500ms = ~1.31s
- Plays every render, including Play Again
- Hover transform doesn't override because hover only kicks in after animation completes (or is debounced by the rule precedence — `:hover` state transforms take over naturally)

### Felt sparkles

On init (after applyTheme), create 8 `<div class="felt-sparkle">` appended to `<body>`:

```js
for (let i = 0; i < 8; i++) {
  const s = document.createElement('div');
  s.className = 'felt-sparkle';
  s.style.top         = `${10 + Math.random() * 80}%`;
  s.style.left        = `${Math.random() * 100}%`;
  s.style.animationDelay = `${Math.random() * 6}s`;
  document.body.appendChild(s);
}
```

CSS:
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

Uses percentage positions so they scale with viewport without a resize listener.

### Title glint

Applied to header `<h1>` and `.ts-title`. Both already use gradient text fill with `-webkit-background-clip: text`. The glint is a pseudo-element overlay with `mix-blend-mode: screen` so it sweeps over the text colour without replacing it.

```css
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

Separate delays so the two titles don't glint in lockstep.

---

## Out of scope (explicitly)

- Session stats / running totals / net £
- Streak tracking
- Real audio samples (synthesized only)
- Mobile haptics
- Admin authentication
- Server-side RNG
- Daily spin / retention mechanics
- Social sharing
- Ambient background music loop
- Prize value currency other than GBP

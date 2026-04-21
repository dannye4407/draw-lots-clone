# Draw Lots

A browser-based lottery reveal game with a full 54-card playing-card deck. An admin configures prize cards and odds; players flip a card to reveal their fate.

## Demo

![screenshot](screenshot.png)

## How to Run

Open `index.html` directly in any modern browser, or use a local server:

```bash
npx serve .
```

**Setup first:** Open `admin.html` to configure prize cards and pool size, then open `index.html` to play.

## Deploy to GitHub Pages

1. Push this repo to GitHub
2. Go to **Settings → Pages → Source: GitHub Actions**
3. Push to `main` — the workflow in `.github/workflows/pages.yml` deploys automatically
4. Live at `https://<username>.github.io/<repo>/`

## How It Works

1. **Admin** sets a pool size (e.g., 10,000) and maps playing cards to prizes with slot counts
   - Example: Ace of Spades = Grand Prize, 1 slot → 1-in-10,000 odds
2. **Player** sees 54 face-down cards and clicks one
3. The RNG (client-side `crypto.getRandomValues`, designed for server-side swap) determines the outcome
4. The card flips to reveal its suit and rank
5. A modal announces the prize — or "No Prize"
6. **Play Again** resets the deck

## Architecture

| File | Role |
|---|---|
| `index.html` | Player game (self-contained, inline CSS + JS) |
| `admin.html` | Admin config panel (self-contained, inline CSS + JS) |
| `.github/workflows/pages.yml` | GitHub Pages auto-deploy |

Config flows `admin.html` → `localStorage` → `index.html`.

The RNG lives in a single `async getOutcome()` function in `index.html`. Replace its body with a `fetch()` call when a real server is ready — no other code changes required.

## Tech Stack

- Vanilla HTML5, CSS3 (custom properties, CSS Grid, 3D transforms)
- ES6 JavaScript (`crypto.getRandomValues` for RNG)
- No frameworks · No build step · No dependencies

## License

MIT — see [LICENSE](LICENSE)

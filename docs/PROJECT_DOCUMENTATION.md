# NEXUS — Core Recalibration

## 1. Executive Summary

NEXUS is a single-page, browser-based puzzle game. The player takes the role of an "operator" reviving a dormant AI reactor by solving rapid-fire number, logic, and pattern puzzles. Each correct answer raises a **stability** meter; each miss drains it. The whole game — markup, styling, and logic — lives in one self-contained HTML file with no server, database, or build step, and is deployed as a static site on GitHub Pages.

Repo: [github.com/jainviyom/game-nexus](https://github.com/jainviyom/game-nexus)
Live: `https://jainviyom.github.io/game-nexus/`

## 2. Business Requirements Document (BRD)

### 2.1 Purpose & Background

The project's goal is a short-session, replayable brain-training game that is trivial to host and share — no accounts, no install, no backend to operate. It doubles as a lightweight portfolio/demo piece for front-end craft: animation, SVG, and game-state design done entirely in vanilla web technology.

### 2.2 Business Objectives

- Deliver an engaging 2–10 minute play loop with a clear sense of progression (score, streak, rank).
- Keep operating cost at zero — static hosting only, no infrastructure to maintain.
- Make the game embeddable/shareable as a single file or a public URL.

### 2.3 Target Users / Personas

| Persona | Motivation |
|---|---|
| Casual player | Wants a quick, low-stakes distraction with visible progress (score/streak). |
| Puzzle enthusiast | Wants increasing challenge — drawn to the "Architect" tier and auto-ramp mode. |
| Recruiter / reviewer | Evaluating front-end craft — clean state machine, animation, accessibility handling. |

### 2.4 Scope — In Scope (delivered)

- Three puzzle categories (Number Core, Logic Grid, Pattern Scan), 13 distinct puzzle generators.
- Three difficulty tiers (Cadet / Operator / Architect) with an optional auto-ramp on core charge.
- Scoring, streak-based combo multipliers, hint system, timed-answer mode.
- Reactor visualization (SVG ring + hex frame) reflecting stability in real time.
- Start screen (difficulty + mode selection) and end-of-session results screen.
- Full keyboard control, responsive layout down to mobile widths, `prefers-reduced-motion` support.

### 2.5 Scope — Out of Scope (current version) / Roadmap

- No persistence: no login, no saved high scores, no server-recorded leaderboard.
- No sound design.
- No analytics/telemetry.
- No automated tests for puzzle generators.

See §9 for a fuller roadmap discussion.

### 2.6 Functional Requirements

1. On load, present a start screen requiring a difficulty choice (Cadet/Operator/Architect) before play begins.
2. Optional toggles, set before starting: **Timed mode** and **Auto-ramp difficulty** (both default on).
3. Each round selects a random category, then a random generator within that category, then renders a prompt and four answer options.
4. Player answers via click or keyboard (`1`–`4`); correct/incorrect state is shown immediately with an explanation.
5. Stability rises on correct answers and falls on wrong answers or timeouts; reaching 100% triggers a bonus "core charge" event, reaching 0% ends the session.
6. A hint control removes two of the three wrong options at a score/stability cost.
7. A results screen reports final score, best streak, puzzles solved, accuracy, and peak stability, and offers a restart.

### 2.7 Non-Functional Requirements

- **Zero backend dependency** — must run from a static file server or `file://`.
- **Performance** — 60fps particle/ring animation on modest hardware; no layout thrashing.
- **Accessibility** — visible focus states on interactive elements; full `prefers-reduced-motion` fallback; keyboard-operable end to end.
- **Responsiveness** — usable from ~320px mobile width up through desktop (breakpoint at 820px).
- **Portability** — single HTML file; only external dependency is a Google Fonts stylesheet.

### 2.8 Assumptions & Constraints

- No persistence layer means progress, best scores, and settings do not survive a page reload — this is an accepted trade-off for the zero-infrastructure goal, not an oversight.
- The two on-screen toggles (Timed mode, Auto-ramp) are read from module-level variables (`optTimed`, `optRamp`) at the moment a difficulty tier is clicked; they are not re-readable mid-session.
- Difficulty tier names (Cadet/Operator/Architect) and rank names (Cadet/Operator/Analyst/Architect/Nexus Prime) intentionally overlap in two labels — a player's displayed *rank* is driven purely by cumulative score, independent of the *tier* they started on.

### 2.9 Success Criteria

- A new player can go from landing on the page to their first puzzle in under 10 seconds with no instructions beyond the start-screen copy.
- A session's outcome (rank, score, accuracy) is legible at a glance on the results screen.
- The page never requires horizontal scrolling or unreadable text at any supported viewport.

## 3. System Architecture

### 3.1 High-Level Architecture

NEXUS has no client/server split — it is a single static document executed entirely in the browser. There is no API layer, no database, and no build pipeline.

```
┌──────────────────────────────────────────────┐
│                Browser (client)                │
│                                                │
│  ┌───────────────┐   ┌───────────────────────┐ │
│  │  Presentation │   │      Game Engine      │ │
│  │  (HTML/CSS)   │◄──┤      (vanilla JS)     │ │
│  │  HUD, reactor │   │  generators, state S, │ │
│  │  SVG, panels, │──►│  round flow, scoring  │ │
│  │  overlays     │   └───────────────────────┘ │
│  └───────────────┘                             │
└──────────────────────────────────────────────┘
                     ▲
                     │ static file serving
                     │
        ┌────────────────────────┐
        │   GitHub Pages (CDN)   │
        │  serves index.html /   │
        │  nexus.html verbatim   │
        └────────────────────────┘
                     ▲
                     │ one external stylesheet request
                     │
        ┌────────────────────────┐
        │      Google Fonts      │
        │  Space Grotesk / Mono  │
        └────────────────────────┘
```

### 3.2 Component Responsibilities

| Component | Responsibility |
|---|---|
| CSS design tokens (`:root` custom properties) | Single source of truth for palette, fonts; drives both the reactor's warn-state recoloring and responsive rules. |
| Reactor SVG (`#reactor`) | Renders hex frame, progress ring (`#ringFill`), glow core, and 36 tick marks; ring stroke-dashoffset is driven by `stab/100`. |
| `GENERATORS` registry | Maps category key (`num`/`logic`/`pattern`) to an array of pure generator functions; `nextRound()` picks a category then a generator at random. |
| Game state `S` | Single mutable object holding score, streak, stability, tier, session flags, and the current puzzle — the entire game's memory. |
| Round flow (`nextRound`, `answer`, `timeout`, `onCorrect`, `onWrong`, `finishRound`) | State machine driving one puzzle from render → input → resolution → next. |
| FX layer (`burst`, `fullBurst`) | Spawns short-lived DOM nodes animated via the Web Animations API for click and core-charge feedback. |
| Overlay screens (`#startScreen`, `#resultScreen`) | Full-screen takeovers for pre-game setup and post-game summary; toggled via a `.hidden` class. |

### 3.3 Round Lifecycle (state machine)

```
resetState() ──► nextRound() ──► [player answers or timer expires]
                     ▲                        │
                     │                        ▼
              finishRound() ◄── onCorrect() / onWrong() / timeout()
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
  stability < 100%           stability >= 100%
  → "Next signal" button     → coreCharged(): +1000, tier
    → nextRound()              may ramp, stability resets to 45
                                → nextRound()

  stability <= 0% at any point → "View report" button → endSession()
```

### 3.4 Deployment Architecture

Static hosting only:

- Source of truth: `main` branch of `github.com/jainviyom/game-nexus`.
- GitHub Pages configured to deploy from `main`, root folder.
- `index.html` and `nexus.html` are identical copies, so the game resolves both at the repo root (`/`) and at the explicit `/nexus.html` path.
- No CI, no environment variables, no secrets — a `git push` to `main` is the entire release process.

### 3.5 Data Flow — One Puzzle Round (detailed)

1. `nextRound()` picks a category and generator, calls it with the current tier, and stores the returned puzzle object on `S.current`.
2. The DOM is rebuilt: category tag, sequence counter, prompt HTML, sub-prompt, and four option buttons (each wired to `answer(btn, opt)`).
3. If timed mode is on, `startTimer()` begins a `setInterval` that shrinks the timer bar and calls `timeout()` if it reaches zero before the player answers.
4. On click or `1`–`4` keypress, `answer()` locks input, stops the timer, disables all options, and visually marks the correct option plus the chosen one (if wrong).
5. `onCorrect()` computes points (base-by-tier × combo multiplier, with a speed bonus in timed mode and a 50% penalty if a hint was used this round), updates streak/stability/peak, fires a particle burst, and either calls `coreCharged()` (stability hit 100) or `finishRound(true)`.
6. `onWrong()` / `timeout()` reset streak, drain stability, and call `finishRound(false)`.
7. `finishRound()` updates the HUD and reveals the "Next signal" (or "View report") button; `Enter` or a click advances to the next round or to `endSession()`.

## 4. Tech Stack

### 4.1 Summary

| Layer | Technology |
|---|---|
| Markup | HTML5 (single document) |
| Styling | CSS3 — custom properties, Grid/Flexbox, `@media` queries, CSS animations |
| Logic | Vanilla JavaScript (ES6+): closures, `Set`, template literals, `Array` methods, no modules |
| Graphics | Inline SVG (reactor hex/ring), DOM-based particle effects |
| Animation | CSS `@keyframes` + the Web Animations API (`Element.animate()`) for particle bursts |
| Fonts | Google Fonts — Space Grotesk (display), Space Mono (mono/labels), loaded via `<link>` with `preconnect` |
| Hosting | GitHub Pages (static) |
| Build tooling | None — no bundler, transpiler, or package manager |
| Persistence | None — in-memory only (`S` object), reset on reload |

### 4.2 Why This Stack

- **No framework, no build step**: the entire game is one file, which keeps hosting to "commit and push" and makes the code trivially portable (double-clickable locally, embeddable anywhere).
- **Inline SVG over canvas** for the reactor: declarative, styleable with CSS variables, and needs no per-frame redraw since only `stroke-dashoffset` and opacity are animated.
- **Web Animations API for particles** instead of a physics/animation library: `element.animate()` gives per-node, self-cleaning (`onfinish` removes the node) transform/opacity tweens with zero dependencies.
- **CSS custom properties** as the palette/typography source of truth, so the single `.warn` class swap (low stability) re-themes the reactor without touching JS.

### 4.3 External Dependencies

There is no package manifest (no `package.json`) and no npm dependencies. The only external network request the page makes is the Google Fonts stylesheet:

```html
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&family=Space+Mono:wght@400;700&display=swap" rel="stylesheet">
```

If served offline or with that request blocked, the page still runs — it falls back to the browser's default sans-serif/monospace fonts.

## 5. Data / State Model

There is no database. All game data is either **generated on the fly** (puzzles) or held in **one in-memory object** (`S`) for the duration of a browser tab session.

### 5.1 Puzzle object shape (return value of every generator)

| Field | Type | Meaning |
|---|---|---|
| `cat` | string | `num` \| `logic` \| `pattern` |
| `prompt` | HTML string | The question, rendered into `#prompt` |
| `sub` | string | One-line hint about what's being asked |
| `options` | array (4) | Answer choices, pre-shuffled |
| `answer` | string/number | The correct option, compared via string equality |
| `explain` | HTML string | Shown after answering, correct or not |

### 5.2 Session state object `S`

| Field | Meaning |
|---|---|
| `tier`, `baseTier` | Current / starting difficulty (0–2); `tier` can rise under auto-ramp |
| `timed`, `ramp` | Session mode flags, fixed at session start |
| `score`, `streak`, `bestStreak`, `solved`, `attempts` | Running scoring counters |
| `stab`, `peak` | Current and highest-seen reactor stability (0–100) |
| `seq` | Round counter, shown as `SEQ 00N` |
| `current` | The active puzzle object |
| `locked` | True between an answer being given and the next round starting (prevents double input) |
| `cores` | Count of full reactor charges (100% reached) this session |
| `timerId`, `timeLeft`, `timeMax` | Timed-mode countdown bookkeeping |
| `hintUsed` | Whether a hint was spent this round (halves points if so) |

## 6. Feature Reference

### 6.1 Puzzle Categories & Generators

| Category | Generators | What they test |
|---|---|---|
| **Number Core** (`num`) | `gMissingOp`, `gMissingNum`, `gPrime`, `gTarget` | Missing arithmetic operator; missing operand in `+`/`−`/`×`; prime identification; multi-step target equation |
| **Logic Grid** (`logic`) | `gTransitive`, `gSyllogism`, `gConditional`, `gParity` | Ordering by chained comparisons ("taller than" chains); syllogism validity (All/Some); conditional reasoning (modus ponens/tollens); odd-one-out by shared property |
| **Pattern Scan** (`pattern`) | `gArithSeq`, `gGeoSeq`, `gFib`, `gSquares`, `gInterleave` | Arithmetic sequences; geometric sequences; Fibonacci-style recurrence; squares/triangular/`n²+1` sequences; two interleaved sequences |

Each generator accepts the current difficulty tier (`0`/`1`/`2`) and scales its own number ranges and step sizes accordingly — difficulty is generator-local, not a global multiplier.

### 6.2 Scoring & Progression

- **Base points per tier**: 100 / 150 / 240 (Cadet / Operator / Architect).
- **Combo multiplier** (by current streak): ×1 below 3, ×2 at 3+, ×3 at 6+, ×4 at 9+.
- **Timed-mode speed bonus**: up to +60% extra, scaled by fraction of time remaining when answered.
- **Hint penalty**: points for that round are halved if a hint was used.
- **Stability gain on correct**: `8 + (multiplier − 1) × 2`.
- **Stability loss on wrong or timeout**: flat `13`.
- **Hint cost**: `40` score and `4` stability; removes 2 of the 3 wrong options; usable once per round.
- **Core charge** (stability reaches 100%): `+1000` bonus, `cores` incremented, stability resets to 45, and — if auto-ramp is on and the current tier is below Architect — the tier increases by one.
- **Ranks** (by cumulative score): Cadet (0) → Operator (600) → Analyst (1600) → Architect (3200) → Nexus Prime (5600).
- **Session ends** when stability reaches 0%, showing final score, best streak, puzzles solved, accuracy (`solved / attempts`), and peak stability.

### 6.3 Controls

| Input | Effect |
|---|---|
| Click / tap an option | Submit that answer |
| `1`–`4` | Submit the corresponding option |
| `Enter` | Advance ("Next signal" / "View report") when available |
| `H` | Request a hint, if available and affordable |

### 6.4 Accessibility & Responsiveness

- All interactive controls (`.opt`, `.btn`, `.diff`, `.toggle`) have a visible `:focus-visible` outline.
- `@media (prefers-reduced-motion: reduce)` collapses all animation/transition durations to near-zero and disables the ambient starfield drift.
- A single breakpoint at `820px` switches the two-column HUD/puzzle layout to a stacked mobile layout: the reactor shrinks and moves into a horizontal strip, the options grid drops to one column, and desktop-only HUD elements (e.g. rank) hide.

## 7. Setup & Deployment Guide

### 7.1 Local development

No install step. Either:

```bash
open index.html         # macOS, opens directly in default browser
```

or serve it (needed if you want the Google Fonts request and any future `fetch` calls to behave exactly as in production):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

There is no build, lint, or test command — edits to `nexus.html`/`index.html` are live on save/reload.

### 7.2 Production (GitHub Pages)

1. Push to `main` on `github.com/jainviyom/game-nexus`.
2. Repo Settings → Pages → **Source: Deploy from a branch**, branch `main`, folder `/ (root)`.
3. GitHub builds and serves the repo root as a static site, typically live within about a minute of the first save.

Live URLs:
- `https://jainviyom.github.io/game-nexus/` (serves `index.html`)
- `https://jainviyom.github.io/game-nexus/nexus.html` (identical copy, explicit path)

### 7.3 Keeping `index.html` and `nexus.html` in sync

The two files are plain copies, not symlinked or templated. Any future edit must be applied to both (or one should be removed and the other left as the canonical file) — there is currently no build step that generates one from the other.

## 8. Known Limitations, Risks & Roadmap Notes

- **No persistence**: refreshing the page loses score, rank, and settings. Adding `localStorage` for best-score/rank would be the natural first enhancement.
- **No backend leaderboard**: all progression is per-session and per-device.
- **Duplicate source files**: `index.html`/`nexus.html` drift risk if only one is edited going forward.
- **No automated tests**: puzzle generators (especially the retry-loop ones like `gMissingOp`) have no unit coverage; a malformed generator could silently degrade (e.g. falling back to a fixed set after 40 failed tries).
- **No sound or haptic feedback.**
- **Single external dependency** (Google Fonts) means the page's typography degrades to system fonts if that request is blocked — acceptable, but not tested against network failure explicitly.

## 9. Appendix

### 9.1 File Structure

```
game-nexus/
├── index.html    # canonical copy served at the Pages root
├── nexus.html    # identical copy, served at an explicit path
└── docs/
    └── PROJECT_DOCUMENTATION.md
```

### 9.2 Glossary

| Term | Meaning |
|---|---|
| **Stability** | 0–100 reactor health meter; the game's core "lives" resource |
| **Core charge** | The event when stability reaches 100%; grants a bonus and may raise difficulty |
| **Combo** | Multiplier applied to points based on current correct-answer streak |
| **Tier** | Difficulty level (0=Cadet, 1=Operator, 2=Architect) — distinct from *rank* |
| **Rank** | Title derived from cumulative score, shown in the HUD (Cadet → Nexus Prime) |
| **Auto-ramp** | Session toggle that raises the tier automatically on each core charge |

### 9.3 Version History

| Commit | Description |
|---|---|
| `d1535da` | Add NEXUS core recalibration puzzle game |
| `de06b23` | Add index.html so GitHub Pages serves the game at the root URL |

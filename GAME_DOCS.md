# 🎲 Yutnori Game — Complete Developer & AI Agent Documentation

> **Purpose:** This document is the single source of truth for any developer or AI agent working on this codebase. It covers architecture, game logic, all implemented features, known bugs, design decisions, and the update changelog. Read this before making any changes.

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack & File Structure](#2-tech-stack--file-structure)
3. [Game Rules & Logic (How Yutnori Works)](#3-game-rules--logic-how-yutnori-works)
4. [Architecture Deep Dive](#4-architecture-deep-dive)
5. [Feature Inventory](#5-feature-inventory)
6. [State Management](#6-state-management)
7. [Board & Coordinate System](#7-board--coordinate-system)
8. [AI Opponent Logic](#8-ai-opponent-logic)
9. [Online Multiplayer (Sound Vision API)](#9-online-multiplayer-sound-vision-api)
10. [Internationalization (i18n)](#10-internationalization-i18n)
11. [Audio System](#11-audio-system)
12. [UI Layout & Responsiveness](#12-ui-layout--responsiveness)
13. [Known Issues & Bugs](#13-known-issues--bugs)
14. [Update Log](#14-update-log)
15. [Planned / Suggested Upgrades](#15-planned--suggested-upgrades)

---

## 1. Project Overview

**Yutnori** (윷놀이) is a traditional Korean board game. This is a **modern web-based multiplayer implementation**. It supports local play, AI play, and online multiplayer via the **Sound Vision API** (`https://yutnori.soundvision.app/api`), including registration, friends, rooms, rematch, and a global wins scoreboard.

- **Frontend:** `index.html` (jammy-style UI)
- **Backend:** `api/` Node.js + Express + Socket.io (JSON file store)
- **Authors:** Arpa Bhowmik (U220), Yeasmin Kabir Keya (U205), Fahima Abida Chowdhury (U210)
- **Purpose:** Academic/educational project

---

## 2. Tech Stack & File Structure

### Technologies
| Layer | Technology |
|---|---|
| Structure | HTML5 (`index.html`) |
| Styling | Vanilla CSS3 |
| Logic | Vanilla JavaScript (ES6+, IIFE) |
| Online Backend | Sound Vision API — Express + Socket.io |
| Data | InfinityFree MySQL (`players`, `friendships`, `rooms`, `room_seats`, `matches`) |
| Realtime | Socket.io path `/api/socket.io` |
| External CDN | Socket.io client 4.8.1 |

### File Structure
```
Yutnori_Game-main/
├── index.html              ← Frontend app (UI + game engine + API client)
├── api/
│   ├── package.json
│   ├── schema.sql          ← MySQL tables (import in phpMyAdmin)
│   ├── .env.example        ← Copy to .env with MYSQL_* credentials
│   ├── README.md           ← Deploy / MySQL / reverse-proxy notes
│   └── src/
│       ├── server.js       ← REST + Socket.io
│       ├── mysql.js        ← MySQL pool + queries
│       ├── db.js           ← IDs, tokens, helpers
│       ├── auth.js
│       └── rooms.js
├── assets/                 ← Team avatars
├── README.md
├── .nojekyll
└── GAME_DOCS.md
```

> Game rules/engine stay in `index.html`. Online identity, friends, rooms, and leaderboard are served by `api/`.

---

## 3. Game Rules & Logic (How Yutnori Works)

### Objective
Each team has **4 pieces**. The first team to move all 4 pieces around the board and finish wins.

### Yut Stick Throw
Four sticks are thrown. Each stick has a flat side and a round side.

| Flat sticks (up) | Result | Steps | Extra Throw? |
|---|---|---|---|
| 0 flat (all round) | **Mo** (모) | 5 | YES |
| 1 flat | **Do** (도) | 1 | NO |
| 2 flat | **Gae** (개) | 2 | NO |
| 3 flat | **Geol** (걸) | 3 | NO |
| 4 flat | **Yut** (윷) | 4 | YES |

> In code: `Math.random() < 0.5` per stick → flat=true (50/50 probability per stick).

### Turn Flow
1. **Throw phase:** Player clicks "Throw the Yut". Result is stored in `state.pending[]`.
2. If result is Yut or Mo → player throws again (bonus throw). Multiple throws accumulate in `state.pending[]`.
3. **Move phase:** Player selects a pending result chip, then clicks a highlighted piece.
4. If a capture occurs → player gets another throw (turn continues).
5. If pending moves remain → player chooses another pending result.
6. Otherwise → turn passes to the next team.

### Movement Rules
- **Stacking:** If your piece lands on a square occupied by your own piece, they stack and move together as a group.
- **Capture:** If your piece lands on a square occupied by an opponent piece, the opponent pieces return to home. The capturing player gets a bonus throw.
- **Finishing:** When a piece's computed destination exceeds the end of the route, `dest()` returns `{finish: true}`, and the piece status becomes `'finished'`.
- **Shortcuts (diagonal paths):**
  - Landing on **N5** (bottom-right corner) → switches to `'low'` route (goes diagonally through A1, A2, Center C, D1, D2).
  - Landing on **N9** (top-right corner) → switches to `'high'` route (goes diagonally through B1, B2, Center C, E1, E2).

### Back-Do (백도) Hidden Rule
- A special traditional rule. When enabled, approximately 1 in 4 "Do" results become **Back-Do** (`steps: -1`).
- On the board, Back-Do moves the piece one square backward.
- From home or the start square (N1), Back-Do places the piece on **N16** (one square before finish).
- Toggle: Settings modal → saved to `localStorage` as `'yutnori-backdo'`.

---

## 4. Architecture Deep Dive

The entire app is an **IIFE (Immediately Invoked Function Expression)** in strict mode:
```js
(()=>{ 'use strict'; /* all code */ })();
```

### Key Global Variables (inside IIFE)
| Variable | Type | Purpose |
|---|---|---|
| `mode` | string | 'local', 'online', or 'ai' |
| `teamCount` | number | 2, 3, or 4 |
| `state` | object | Full game state (see section 6) |
| `lang` | string | Current language: 'en', 'bn', 'ko' |
| `soundEnabled` | boolean | Whether sound effects are on |
| `backDoEnabled` | boolean | Whether Back-Do rule is active |
| `isHost` | boolean | Online mode: is this user the room host? |
| `roomId` | string | Online mode: current room code |
| `myTeamIndex` | number | Online mode: which team index this client controls |
| `setupPlayers` | array | 4-element array of player config for local/online |
| `aiSetupPlayers` | array | 2-element array for AI mode (Player + AI) |
| `firebaseReady` | Promise | Firebase init promise (singleton) |

### Page Sections (show/hide via CSS `hidden` class)
| Section ID | Description |
|---|---|
| `#landing` | Home screen with game mode selection |
| `#setup` | Player configuration screen |
| `#game` | Main game board screen |

### Modals
| Modal ID | Content |
|---|---|
| `#rulesModal` | How to Play rules |
| `#settingsModal` | In-game settings (name/photo/Back-Do) |
| `#winnerModal` | Winner announcement overlay |

---

## 5. Feature Inventory

### Implemented and Working
- Local multiplayer (2, 3, or 4 teams on one device)
- AI mode (single player vs. greedy computer)
- Online multiplayer via Firebase Realtime Database
  - Create room (host), join room via code or URL
  - Real-time state sync, host-only game start
  - Disconnect handling via `onDisconnect().set(false)`
- 3-language i18n: English, Bangla, Korean (persisted to localStorage)
- Custom player avatars (upload JPG/PNG, saved as base64 in localStorage)
- Custom player names (editable in setup and in-game settings)
- Sound effects via Web Audio API (throw, move, capture, win) with on/off toggle
- Back-Do toggle UI (logic not wired)
- Move preview: animated dashed path + glowing destination dot + step label
- Piece tooltips on hover showing destination
- Stacking: friendly pieces stack and move together (shown with badge)
- Capture: landing on opponent sends their pieces home and grants bonus throw
- Shortcuts: diagonal routes via N5 (low) and N9 (high) corners through center
- Finished pieces dock (left sidebar with avatar icons)
- Game log (scrollable, last 80 entries)
- Score panel (all teams' progress with current team highlighted)
- Winner detection and modal
- New Game / Play Again reset
- Responsive design (650px, 1050px, 1200px breakpoints)
- 3D board aesthetic with CSS perspective, shadows, wood-grain look

### Partially Implemented
- Online room-full detection: Firebase transaction works but UX feedback is minimal

### Not Implemented
- True online spectator mode
- Turn timer / time limit per turn
- Game history persistence (page refresh loses the game)
- Real recorded sound effects (only synthesized beeps)
- Mobile swipe gestures

---

## 6. State Management

All game state lives in the `state` object:

```js
state = {
  players: [
    {
      name: "Blue Team",
      photo: "data:image/..." or "assets/team-blue.png",
      color: "#2379a7",
      ai: false,
      pieces: [
        { id: "0-0", status: "home" | "board" | "finished", route: "outer" | "low" | "high", index: -1 to 16 },
        { id: "0-1", ... },
        { id: "0-2", ... },
        { id: "0-3", ... }
      ]
    }
    // up to 4 players
  ],
  current: 0,         // Index of current player's turn
  phase: "throw" | "move" | "junction",
  pending: [          // Throw results not yet used for a move
    { name: "Yut", steps: 4, bonus: true, flat: [true,true,true,true], id: "abc123" }
  ],
  selected: null,     // Currently selected throw result (or a pending item)
  gameOver: false,
  logs: [...],       // Max 80 entries
  finishOrder: [],   // Team indices in the order they finished all 4 pieces
  junctionPending: null // { pieceId, steps, node, rollId } while phase === 'junction'
}
```

### Piece Status Values
| Status | Meaning |
|---|---|
| 'home' | Not yet entered the board (or captured back home). index = -1 |
| 'board' | On the board at routes[route][index] |
| 'finished' | Completed the full circuit |

### State Persistence
- State is NOT persisted to localStorage (page refresh loses the game).
- In online mode, state is synced to Firebase under `rooms/{roomId}/state`.

---

## 7. Board & Coordinate System

### Coordinate Space
All positions are expressed as **percentage of board width/height** (0–100%). The board is a square `div#board`.

### Named Nodes
```
N1:[84,82]   N2:[68,82]   N3:[52,82]   N4:[36,82]   N5:[20,82]   <- Bottom row
N6:[20,65.5] N7:[20,49]   N8:[20,32.5] N9:[20,16]                <- Left col
N10:[36,16]  N11:[52,16]  N12:[68,16]  N13:[84,16]               <- Top row
N14:[84,32.5] N15:[84,49] N16:[84,65.5] N17:[84,82]              <- Right col

Shortcut nodes:
A1:[30.67,71]  A2:[41.33,60]  C:[52,49]  D1:[62.67,38]  D2:[73.33,27]
B1:[30.67,27]  B2:[41.33,38]  E1:[62.67,60]  E2:[73.33,71]
```

> N1 and N17 are the SAME physical location (bottom-right = start/finish). N1 is never rendered as a visible node.

### Routes
```
outer: N1 → N2 → N3 → N4 → N5 → N6 → N7 → N8 → N9 → N10 → N11 → N12 → N13 → N14 → N15 → N16 → N17
low:   N1 → N2 → N3 → N4 → N5 → A1 → A2 → C  → D1 → D2  → N13 → N14 → N15 → N16 → N17
high:  N1 → N2 → N3 → N4 → N5 → N6 → N7 → N8 → N9 → B1  → B2  → C   → E1  → E2  → N17
```

### Corner & Special Nodes
| Node | Position | CSS Class | Route Switch |
|---|---|---|---|
| N5 | Bottom-right corner | .corner | Triggers 'low' route shortcut |
| N9 | Top-right corner | .corner | Triggers 'high' route shortcut |
| N13 | Top-left corner | .corner | No switch |
| N17 | Start/Finish | .corner | Finish line |
| C | Center | .center | No switch (shared by low and high paths) |

### Home Positions
```js
function homePos(pi, idx) {
  const bases = [[20,8.5],[40,8.5],[60,8.5],[80,8.5]]; // x% per team (0-3)
  const offsets = [[-3.0,-1.9],[3.0,-1.9],[-3.0,1.9],[3.0,1.9]]; // 2x2 grid per team
}
```
Pieces at home are displayed above the board (y ≈ 8.5%) in a 2×2 cluster per team.

---

## 8. AI Opponent Logic

The AI is **very simple (greedy)**, running via `aiTurn()`:

```
1. If phase = 'throw'  → call toss() with 700ms delay
2. If phase = 'move':
   a. Take the first pending result (state.pending[0])
   b. Priority: piece that FINISHES > piece ON BOARD > any piece
   c. Move that piece with 500ms delay
```

The AI does NOT consider: capturing opponents, avoiding capture, optimal shortcut routing, or stacking strategy.

**AI mode colors:** Player = `#ff5fa2` (pink), AI = `#7a1538` (dark maroon). Different from standard 4-color palette.

---

## 9. Online Multiplayer (Sound Vision API)

Firebase has been **removed**. Online play uses `https://yutnori.soundvision.app/api` (local dev: `http://localhost:8787/api`).

### Identity
- First launch: register with **display name only**
- Server returns `playerId`, Friend ID (`YUT-XXXX`), `sessionToken`
- Stored in `localStorage` key `yutnori-player`
- Name editable in Settings; Friend ID is permanent

### Friends
- Add by Friend ID → pending request → accept/decline
- Presence: `offline` | `idle` | `in_room` | `playing`
- Host can invite a friend into an empty seat in the **waiting lobby only**

### Rooms
- Create: pick 2/3/4 max seats → waiting screen with room code + share link
- Join: invite code or `?room=CODE`
- Host may **start early** with 1+ seated players; empty seats stay empty for that match (no mid-match join)
- Invite code remains usable for waiting/rematch lobby until room closes
- Rematch: Play Again / Leave votes (Ludo-style); decliners free seats; all accepters return to waiting

### Realtime
- Socket.io (`/api/socket.io`) events: `room:update`, `game:state`, `game:action`, `game:over`, `rematch:vote`
- `broadcast()` emits full game state after throws/moves

### Leaderboard
- `GET /api/leaderboard` — global wins / games
- Wins increment when an online match finishes (`game:over` path records match)

### Run API locally
```bash
cd api && npm install && npm start
```
See `api/README.md` for reverse-proxy deploy to `yutnori.soundvision.app`.

---

## 10. Internationalization (i18n)

### Supported Languages
| Code | Language | Status |
|---|---|---|
| en | English | Full |
| bn | Bangla (বাংলা) | Full |
| ko | Korean (한국어) | Full |

### How It Works
- `I18N` is a hardcoded object with all keys for all 3 languages
- `t(key)` returns `I18N[lang][key]` with fallback to English
- `applyLanguage()` updates all DOM text nodes
- Language saved to `localStorage` as `'yutnori-language'`
- Language selector on both landing (`#landingLang`) and game (`#gameLang`) screens

### Key Translation Strings
`localMatch`, `onlineMatch`, `aiMatch`, `play`, `howToPlay`, `back`, `chooseHelp`, `startMatch`, `throwYut`, `throwPrompt`, `chooseResult`, `chooseAnother`, `capturedAgain`, `availableMoves`, `gameLog`, `finishedPieces`, `currentTurn`, `goal`, `goalText`, `throwValues`, `throwText`, `move`, `moveText`, `close`, `gameSettings`, `backDo`, `cancel`, `save`, `completed`, `playAgain`, `wins`, `captured`, `moved`, `throwAgain`, `capture`, `finish`, `steps`, `step`, `team`, `friends`, `scoreboard`, `waitingRoom`, `registerTitle`

---

## 11. Audio System

Uses **Web Audio API** — no external audio files, all synthesized.

| Event | Frequency | Effect |
|---|---|---|
| throw | 180Hz → 420Hz | Rising tone |
| move | 330Hz | Short beep |
| capture | 145Hz | Low thud |
| win | 520Hz → 880Hz | Rising celebration |

- Toggle: `#soundBtn` button in game topbar
- State: `localStorage['yutnori-sound']` = 'true' or 'false'
- Button turns gold when sound is ON

---

## 12. UI Layout & Responsiveness

### Color Palette (CSS Custom Properties) — jammy.fun-inspired (v1.4)
```css
:root {
  --bg:     #0d1713;   /* Night background */
  --panel:  #17211b;   /* Panel */
  --panel-deep: #0d1511;
  --cream:  #ead8b2;   /* Parchment text */
  --parchment-bright: #fff0cf;
  --gold:   #d9ba70;   /* Accent */
  --muted:  #a89b80;   /* Dim parchment */
  --blue:   #2379a7;
  --red:    #d64a36;
  --green:  #3e955a;
  --purple: #8a63d2;
  --font-ui: "Pretendard Variable", Pretendard, sans-serif;
  --font-display: MaruBuri, Batang, serif;
}
```

### UI Shell (v1.4 jammy twin)
- Lobby: centered brand + mode list (Local / Online / AI)
- Game: top HUD (team avatars + score dots), center CSS board with on-board yut sticks, bottom action dock
- Dock: throw button, Piece 1–4 buttons (with your `assets/team-*.png` thumbnails), junction route cards
- Animations: throw-result overlay, piece CSS tween, victory confetti
- **Piece art preserved:** team PNGs + custom uploads (not replaced by plain discs)

### Responsive Breakpoints
| Breakpoint | Layout |
|---|---|
| > 1200px | Full desktop: dock=176px, board=735px, side=350px |
| 1051–1200px | Narrower dock (158px), smaller board (670px), side=326px |
| < 1050px | Single column; dock becomes 2-column grid |
| < 650px | Mobile: everything single column; pieces/nodes scale up |
| max-height:780px + min-width:1051px | Compact for short landscape viewports |

### CSS Architecture Notes
- **Two `<style>` blocks in `<head>`:**
  1. Main styles — lines 3 to 31 of index.html
  2. `id="finished-dock-alignment-fix"` — Desktop alignment refinements — lines 32 to 112
- Team color CSS var: `--team` (on `.player-card`), `--turn` (on `.turn-card`)
- Animations: `@keyframes bob` (bobbing pieces), `@keyframes routePulse` (move preview fade), `@keyframes dashMove` (animated dash on route)

---

## 13. Known Issues & Bugs

| ID | Severity | Area | Description | Status |
|---|---|---|---|---|
| BUG-001 | MEDIUM | Game Logic | Back-Do rule not wired up. Toggle UI exists and saves to localStorage but `dest()` and `movePiece()` do not implement backward movement. | **FIXED** — Back-Do throws (`steps:-1`) move board pieces back one space; from home/start they go to N16 |
| BUG-002 | LOW | Assets | `team-gold.png` is unused. File exists in /assets/ but is not referenced in code. | MINOR |
| BUG-003 | MEDIUM | Online Mode | No host migration. If host disconnects, other players are stuck with no way to restart or promote a new host. | **FIXED** — lowest online `teamIndex` player is promoted via `maybeMigrateHost()` |
| BUG-004 | LOW | Online Mode | Player photos (base64 up to 300KB) are included in every Firebase `broadcast()`. Can cause large payloads. | OPEN |
| BUG-005 | LOW | UI | Piece z-index on mobile. On very small screens, stacked piece badges may overlap adjacent pieces. | OPEN |
| BUG-006 | LOW | AI Mode | AI avatar SVG labels stay in the language they were generated in, even if user switches language mid-game. | OPEN |
| BUG-007 | DESIGN | Game Logic | Pieces arriving at Center (C) via 'low' route and 'high' route share the same coordinate key. They will stack/capture each other. This is CORRECT per official Yutnori rules. | BY DESIGN |
| BUG-008 | LOW | UI | Winner modal shows only 1st place. In 3-4 team games, no 2nd/3rd place ranking is displayed. | **FIXED** — `finishOrder` + podium medals in winner modal |

---

## 14. Update Log

### v2.0.0 — Sound Vision social API
**Date:** 2026-07-27

**What changed:**
- Replaced Firebase online multiplayer with Sound Vision API (`api/` Express + Socket.io)
- One-time name registration + permanent Friend ID
- Friends list with request/accept and presence (idle / in room / playing)
- Waiting lobby with early host start and empty seats locked during match
- Ludo-style rematch votes; global wins scoreboard
- Settings: change display name + copy Friend ID
- Docs: `api/README.md` + this section updated

---

### v1.0.0 — Initial Release
**Date:** Pre-2026-07-27
**Authors:** Arpa Bhowmik, Yeasmin Kabir Keya, Fahima Abida Chowdhury

**What was built:**
- Complete Yutnori game in a single index.html file
- Local multiplayer (2–4 teams)
- AI opponent (simple greedy logic)
- Online multiplayer via Firebase Realtime Database
- 3-language support: English, Bangla, Korean
- Custom player names and avatar photo uploads
- Web Audio API sound effects
- 3D-styled board with SVG route lines
- Animated move preview (dashed path + glowing dot)
- Piece stacking and capture mechanics
- Shortcut routes via corners and center
- Finished pieces dock panel
- Settings modal with Back-Do toggle (UI only, logic pending)
- Responsive CSS for desktop, tablet, mobile
- GitHub Pages deployment

---

### v1.1.0 — Desktop Layout Alignment Fix
**Date:** Pre-2026-07-27
**Author:** Unknown (original team)

**What changed:**
- Added second `<style>` block with `id="finished-dock-alignment-fix"` (lines 32–112 of index.html)
- Refined desktop grid: dock fixed at 176px width, flex-direction column
- `.slots` changed to 4-column CSS grid (2×2 avatar layout for finished pieces)
- Board wrap max-width adjusted per viewport
- Added 1050px–1200px intermediate breakpoint

**Why:** The Finished Pieces dock was misaligned or overflowing on certain desktop viewport sizes.

---

### v1.4.0 — Jammy.fun Visual/UX Twin
**Date:** 2026-07-27

**What changed:**
- Restyled lobby + game chrome to match jammy.fun/yut (Pretendard/MaruBuri, night/parchment/gold tokens)
- Top HUD with team images + score dots; bottom action dock with throw / piece buttons / route cards
- Throw-result overlay, piece move tween, victory confetti
- **Kept** `assets/team-*.png` and custom photo uploads on pieces/HUD/dock
- Game logic (junction, Back-Do, online, AI) unchanged

---

### v1.3.0 — Bug Fixes & Junction Choice (implementation_plan.md)
**Date:** 2026-07-27

**What changed:**
- Fixed critical syntax corruption (`assets` / `I18N` / `defaultNames`) that prevented the game from loading
- Junction choice dialog at N5, N9, and center (C): Shortcut vs Outer/Diagonal path
- Back-Do movement wired in `dest()` (BUG-001)
- Winner podium ranking via `finishOrder` (BUG-008)
- Online host migration when host disconnects (BUG-003)
- Auto-select single pending move; home pieces in a neat row; AI random junction choice
- i18n keys for junction prompts (en / bn / ko)

---

### v1.2.0 — GAME_DOCS.md Created
**Date:** 2026-07-27
**Author:** AI Agent (Antigravity)

**What changed:**
- Created GAME_DOCS.md as comprehensive reference for all future developers and AI agents
- No changes to index.html

---

## 15. Planned / Suggested Upgrades

| Priority | Feature | Notes |
|---|---|---|
| HIGH | Wire up Back-Do rule | `dest()` needs handling for negative step when `steps===1` and piece is at home. Define what "back" means on the outer route. Closes BUG-001. |
| HIGH | AI difficulty levels | Add 'easy' (random), 'medium' (current greedy), 'hard' (heuristic with capture awareness). |
| MEDIUM | Turn timer | Optional countdown per turn, configurable in Settings. |
| MEDIUM | Host migration | If host disconnects in online mode, promote next connected player to host. Closes BUG-003. |
| MEDIUM | Reduce Firebase photo payload | Strip photos from state broadcast. Store photos only in `setup`, not replicated in `state.players[i].photo` on every broadcast. Closes BUG-004. |
| MEDIUM | Winner ranking (full podium) | Track finish order and show 1st/2nd/3rd place in winner modal. Closes BUG-008. |
| MEDIUM | Add 4th language | Japanese or Chinese given the East Asian context of the game. |
| LOW | Animated piece movement | Pieces currently teleport. Add CSS transition along route path segments. |
| LOW | Real sound effects | Replace synthesized beeps with recorded Yut stick sounds. |
| LOW | Game history persistence | Save completed game results to localStorage. |
| LOW | Remove unused team-gold.png | Either use it (rename Purple to Gold) or delete it. Closes BUG-002. |
| LOW | Separate CSS/JS files | Refactor monolithic index.html into index.html + style.css + game.js. |
| LOW | PWA support | Add manifest.json + service worker for offline play. |

---

## Quick Reference for AI Agents

**Before making any change, read this section.**

1. **All code is in `index.html`** — no separate CSS or JS files exist.
2. **The JS is minified-style** — lines are very long (some lines are entire screens of code). Use line numbers carefully.
3. **CSS is split across two `<style>` blocks** — main styles at lines 3–31, desktop alignment fix at lines 32–112.
4. **State object is the source of truth** — always update `state` then call `render()` and `broadcast()`.
5. **`buildBoard()`** — call once when starting a game. Creates DOM nodes for board. Do NOT call repeatedly.
6. **`render()`** — refreshes all UI from state. Internally calls `renderPieces()` and `renderLog()`.
7. **`broadcast()`** — sends state to Firebase in online mode. Always call after state mutations.
8. **`t(key)`** — always use this for any user-facing string so it is translated correctly.
9. **localStorage keys:**
   - `'yutnori-language'`   → 'en' | 'bn' | 'ko'
   - `'yutnori-sound'`      → 'true' | 'false'
   - `'yutnori-backdo'`     → 'true' | 'false'
   - `'yutnori-photos-v3'`  → JSON array of 4 base64 photo strings
10. **Team colors array (index matches state.players index):**
    - 0: '#2379a7' (blue)
    - 1: '#d64a36' (red)
    - 2: '#3e955a' (green)
    - 3: '#8a63d2' (purple)
11. **Asset paths:** `assets/team-blue.png`, `assets/team-red.png`, `assets/team-green.png`, `assets/team-purple.png`
12. **AI mode uses different colors:** Player = '#ff5fa2', AI = '#7a1538'

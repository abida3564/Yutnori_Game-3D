# Yutnori Game — Bug Fixes & Improvements

## What I observed from the reference game (jammy.fun/yut)

Key UX differences between the current game and the reference:

1. **Shortcut/Junction Choice** — When a piece lands exactly on N5 or N9, the reference game ASKS the player "Which way at this junction?" with "Shortcut" vs "Outer path" buttons. Our game auto-forces the shortcut without asking.
2. **Piece selection via bottom bar** — The reference shows piece buttons (Piece 1, Piece 2...) with status below instead of clicking directly on board pieces.
3. **Move preview** — Path highlighted with animated glow showing destination "Mo lands here" tooltip.
4. **Home piece layout** — Pieces waiting at home are shown above the board as colored discs in a row.
5. **Bottom action bar** — Clean bottom panel: left = current turn info, right = action button.
6. **Score dots in topbar** — Small circles (o o o o) next to each team name show piece status.
7. **Yut sticks on board** — Physical yut sticks rendered on the board itself after a throw.

## Bugs to Fix + Features to Implement

### BUG-001 (MEDIUM) — Back-Do Rule Logic
Wire up the Back-Do hidden rule in `dest()` and `movePiece()`.

### BUG-003 (MEDIUM) — No host migration
Add basic handling for when the online host disconnects.

### BUG-008 (LOW) — No finish ranking in winner modal
Track and display 2nd/3rd place finish order.

### CRITICAL UX FIX — Junction / Shortcut Choice Dialog
Currently when a piece lands on a corner node (N5 or N9), it auto-takes the shortcut. Per official Yutnori rules, the PLAYER must CHOOSE which path to take. This is a major gameplay bug.

### UX FIX — Home piece positioning (visual)
Pieces at home are displayed above the board with offsets that can overlap. Clean this up to match a neat row layout.

### UX FIX — Turn indicator improvements
Show a cleaner current-turn indicator with team color dot, like the reference.

### UX FIX — Status messages clarity
Improve status messages to be clearer (e.g., "Which way at the junction?" when at a corner).

### UX FIX — Pending move chips usability
Make chips more obvious / easier to use. Auto-select when only one pending move exists.

## Proposed Changes to index.html

### 1. Junction Choice System (most critical gameplay fix)
- Add a `'junction'` phase to the game state
- When `dest()` detects a piece would land on N5 or N9, instead of auto-switching route, return `{junction: true, node: 'N5'|'N9', ...}` 
- In `movePiece()`, detect junction and enter `phase='junction'` storing the pending piece id and step count
- Add junction choice UI: two buttons "Shortcut" and "Outer path" shown in the bottom action area
- `resolveJunction(choice)` function applies the chosen route

### 2. Back-Do rule (BUG-001)
- In `dest()`, when `backDoEnabled && steps===1 && piece.status==='home'`, return `{route:'outer', index: routes.outer.length-2, node:'N16'}` (last node before finish = N16, moving backwards from start)

### 3. Winner ranking (BUG-008)
- Add `state.finishOrder = []` array
- When a team finishes, push their index to `finishOrder`
- Winner modal shows 🥇 🥈 🥉 for top 3

### 4. Auto-select single pending move
- When only one pending result exists, automatically select it so the player only needs to click a piece

### 5. Score dots in header
- Small colored circles (○ for unfinished, ● for finished pieces) next to team name in topbar

### 6. i18n additions
- Add new translation keys for junction prompts in all 3 languages

## Verification Plan
- Open index.html in browser
- Test Local Match: verify junction choice appears at N5 and N9
- Test Back-Do: enable it in settings, verify a piece at home with a "Do" roll moves backward
- Test 3-team game to verify finish ranking order
- Verify all 3 languages show the junction prompt correctly
- Test AI mode (AI should randomly choose shortcut or outer at junctions)

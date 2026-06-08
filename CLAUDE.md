# CLAUDE.md

Behavioral guidelines and project context for Claude Code on the Shengji scorecard project.

---

## General Principles

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan before implementation, then verify each step.

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## Project Context: Shengji Scorecard

### What This App Does

A mobile-first web app for recording and analyzing six-player (and five-player) card game sessions for 升级 (Sheng Ji / Upgrade). Self-contained single HTML file (`index.html`), no build step, no dependencies except Google Fonts.

### Current Task

**Add support for 5-player mode with different rules:**

1. **Dealer helper rules**: Max 1 helper (vs. 6-player which allows 0+)
2. **Red five penalties**: Bidirectional (Farmer's captured red fives now penalize Farmer side too, vs. 6-player which is one-directional)
3. **Achievements**: Remove "浪漫双人车" series (romantic 2-person car driver/passenger) in 5-player mode, as it's not rare
4. **Configuration**: Session-level `gameMode: '5player' | '6player'` to branch logic accordingly

### Code Architecture

- **Single HTML file**: All HTML, CSS, and JS in `index.html` (~2000 lines)
- **State object**: Persisted to `localStorage` key `sj_v3`
  - `state.players`: 6 hardcoded players (id, name, avatar, seat)
  - `state.sessions`: Array of archived sessions
  - `state.currentSession`: Active session object
- **Session object** includes:
  - `gameMode`: NEW — will add this for 5/6 distinction
  - `players`: Array of player IDs in this session
  - `rounds`: Array of round objects
  - `playerLevels`: Current level per player
- **Round object** includes scoring, red five captures, penalties, achievements
- **Key functions**:
  - `getScoreResult(score)` — determines level changes
  - `getScoreWinner(score)` — determines win/loss for stats
  - `computeAchievements(round, session)` — generates per-round achievements
  - `saveState()` — persists to localStorage

### Game Rules (Both Modes)

**Level system:**
- 11 levels per round: `2, 4, 6, 7, 8, 9, 10, J, Q, K, A`
- 5 main rounds: R1 → R5
- Basement rounds: B1 → B10 (below R1)
- Full flat sequence: B10:2 ... R5:A

**Scoring → Level changes:**
| Farmer Score | Result |
|---|---|
| 0 | Dealer +3 |
| 5–55 | Dealer +2 |
| 60–115 | Dealer +1 |
| 120–175 | **Farmer win** (0 levels, counts as farmer win) |
| 180–235 | Farmer +1 |
| 240–295 | Farmer +2 |
| 300+ | Farmer +3 |

**Red five penalties (applied to dealer side in 6-player):**
- ♥5 captured: −2 levels per card
- ♦5 captured: −1 level per card

**In 5-player mode, red five penalties are bidirectional:**
- If Farmer captures Dealer's red fives → Dealer loses levels (as before)
- If Dealer captures Farmer's red fives → Farmer loses levels (NEW)

**Dealer rotation:**
- Dealer stays if dealer side wins (score ≤ 115 net of penalties)
- Next player clockwise becomes dealer if farmer wins (score ≥ 120)

### Implementation Notes

**Where to branch on gameMode:**

1. **Helper validation** (`renderScorePage` or wherever helpers are selected)
   - 6-player: Allow 0+ helpers
   - 5-player: Allow max 1 helper

2. **Red five penalty logic** (in `getScoreResult` or similar)
   - 6-player: Only apply to dealer side
   - 5-player: Apply bidirectionally based on who captured

3. **Achievement computation** (in `computeAchievements`)
   - 6-player: Include 浪漫双人车 driver & passenger
   - 5-player: Skip those two achievements

4. **UI** (grid layout for players)
   - Adjust based on player count (5 vs 6 columns)

**Code style in index.html:**
- Minified CSS in `<style>` block, JS inline in `<script>`
- No comments except where logic is non-obvious
- Variables/functions use camelCase
- Conditional branches use ternary or simple if/else
- State mutations followed by `saveState()`

### Success Criteria

- [ ] User can create a session and choose "5-player" or "6-player" mode
- [ ] 5-player sessions enforce max 1 helper
- [ ] 5-player sessions apply red five penalties bidirectionally
- [ ] 5-player sessions exclude 浪漫双人车 achievements
- [ ] 6-player sessions work as before (regression test)
- [ ] Existing sessions (pre-update) still load and work

---

## Deployment & Testing

- **Hosted**: GitHub Pages, `main` branch root → https://jimyuhaowu.github.io/shengji/
- **Local testing**: Open `index.html` in a browser; uses localStorage
- **Data**: Persisted in localStorage key `sj_v3`; can export/import JSON in Settings tab
- **No CI/tests**: This is a single-file app; manual testing in browser is the norm

---

## Known Limitations / Future Ideas

- Multi-device sync (previously attempted, removed)
- Dark/light theme toggle
- Round undo (currently editing collapses subsequent rounds)
- Configurable player count (was hardcoded to 6; now will support 5/6 explicitly)

# 升级 记分册

A mobile-first web app for recording and analysing six-player 拖拉机 (Sheng Ji / Upgrade) card game sessions. Hosted at **https://jimyuhaowu.github.io/shengji/**.

---

## Tech Stack

Single self-contained HTML file (`index.html`) with no build step and no dependencies except:
- Google Fonts (Noto Serif SC, Noto Sans SC, Playfair Display) — loaded via CDN
- All game logic, state, rendering, and styling in one file

---

## Project Structure

```
index.html          — entire app (HTML + CSS + JS)
README.md           — this file
```

---

## Game Rules Implemented

### Level System
- 11 levels per round: `2, 4, 6, 7, 8, 9, 10, J, Q, K, A` (3, 5, Joker excluded)
- Multiple rounds: R1 → R2 → R3 → R4 → R5
- Basement rounds below R1: B1 (地下1) → B2 → ... → B10 (deepest)
- Full flat sequence: `B10:2 ... B1:2, R1:2, R1:4 ... R1:A, R2:2 ...`
- All level movement is bidirectional — basement rounds are fully traversable

### Scoring → Level Changes
| Farmer Score | Result |
|---|---|
| 0 | Dealer side +3 levels |
| 5–55 | Dealer side +2 levels |
| 60–115 | Dealer side +1 level |
| 120–175 | **Farmer win, 0 levels** (counts as farmer win in stats) |
| 180–235 | Farmer side +1 level |
| 240–295 | Farmer side +2 levels |
| 300+ | Farmer side +3 levels |

### Red Five Penalties (on top of score result, always applied to dealer side)
- ♥5 captured by farmers: Dealer side −2 levels **per card** (up to 3 cards)
- ♦5 captured by farmers: Dealer side −1 level **per card** (up to 3 cards)

### Wrong-Play Penalties
- Individual player −1 level per wrong play
- Multiple penalties per player per round are allowed

### Dealer Rotation
- Dealer side wins (score ≤ 115, net of red five penalties): Dealer stays
- Farmer side wins (score ≥ 120): Next player clockwise becomes Dealer

---

## Data Model

### State Object (persisted to `localStorage` key `sj_v3`)

```js
state = {
  players: [
    { id: 0, name: "BR", avatar: "data:image/jpeg;base64,...", seat: 0 },
    // ... 6 players total
  ],
  sessions: [ /* archived Session objects */ ],
  currentSession: Session | null,
}
```

### Session Object

```js
{
  id: timestamp,
  startTime: timestamp,
  endKey: "R2:A",           // level key that ends the session when any player reaches it
  currentDealerId: 0,
  players: [0,1,2,3,4,5],  // player ids in this session
  playerLevels: { 0: "R1:4", 1: "B1:2", ... },
  initialLevels: { 0: "R1:2", ... },
  rounds: [ Round, ... ],
}
```

### Round Object

```js
{
  id: 0,                          // index within session
  dealerId: 2,
  helperIds: [4],
  score: 150,
  heartFiveCaptors: [1, 3],       // farmer pids who captured each ♥5 (length = count captured)
  heartFiveVictims: [2, 4],       // dealer-side pids each ♥5 was taken from
  diamondFiveCaptors: [0],
  diamondFiveVictims: [2],
  penalties: { 0: 0, 1: 1, 2: 0, 3: 0, 4: 0, 5: 0 },  // wrong-play counts per player
  achievements: [ { name: "小跳", color: "green", pid: 1 }, ... ],
  levelsBefore: { 0: "R1:2", ... },
  levelsAfter: { 0: "R1:4", ... },
  changes: { 0: 1, 1: 0, ... },  // net level delta per player
  timestamp: timestamp,
}
```

### Level Key Format
`"<round>:<level>"` — e.g. `"R1:A"`, `"B2:2"`, `"R3:10"`

---

## Key Functions

### Level Logic (`LEVEL_SEQ`, `stepLevel`, `keyIndex`)
- `LEVEL_SEQ` — flat ordered array of all level keys from B10:2 to R5:A
- `stepLevel(key, steps)` — move a player up/down the sequence, clamped at ends
- `keyIndex(key)` — get numeric index for comparison/charting

### Score Logic
- `getScoreResult(score)` — returns `{side, levels}` for **level change computation**
- `getScoreWinner(score)` — returns `'dealer'|'farmer'` for **win/loss statistics** (120–175 = farmer)

### Achievements (`computeAchievements`, `computeSessionAchievements`)
Each achievement: `{ name: string, color: 'gold'|'red'|'green'|'purple', pid: number|null }`

#### Per-Round Achievements
| Achievement | Trigger | Assigned to |
|---|---|---|
| 大光头 | Score = 0 | All dealer side |
| 小光头 | Score 5–59 | All dealer side |
| 小跳 | Score 180–239 | All farmer side |
| 大跳 | Score 240–299 | All farmer side |
| 超级跳 | Score 300+ | All farmer side |
| 抓住红五♥ | ♥5 captured | Captor (farmer) |
| 被抓红五♥ | ♥5 captured | Victim (dealer side) |
| 抓住方五♦ | ♦5 captured | Captor (farmer) |
| 被抓方五♦ | ♦5 captured | Victim (dealer side) |
| 坏了坏了 | Red five captured AND dealer lost | All dealer side |
| How dare you? | Dealer called 0 helpers | Dealer |
| Lone Wolf | Dealer called 0 helpers AND won | Dealer |
| 浪漫双人车驾驶员 | Dealer called exactly 1 helper | Dealer |
| 浪漫双人车乘客 | Dealer called exactly 1 helper | The helper |
| 手残党 | Player got ≥2 wrong-play penalties | That player |
| 虚惊一场 | Got penalty but level unchanged | That player |
| 房东收租 | Dealer wins 3rd consecutive round | Dealer |
| 不倒翁 | Dealer wins 5th consecutive round | Dealer |
| 一个人玩儿得了 | Dealer wins 7th consecutive round | Dealer |
| 我要验牌 | Dealer wins 9th consecutive round | Dealer |
| Redemption Arc | Player levels up from basement to R1 | That player |

#### Per-Session Achievements
| Achievement | Trigger |
|---|---|
| 大地主 | Most rounds as Dealer |
| 好帮手 | Most rounds as Helper |
| 老农民 | Most rounds as Farmer |
| 过客 | Was Dealer but never held it consecutively |

---

## Pages & Navigation

| Tab | Content |
|---|---|
| 主页 | Active session: current levels, progress, recent rounds, session achievements |
| 记分 | Round entry form: role assignment, score slider, red five prompts, penalties |
| 统计 | **Current session only**: win rates, level trend chart, achievement counts |
| 生涯 | **All sessions**: career win rates, average scores, lifetime achievements |
| 设置 | Player names/avatars, seat order (▲▼ buttons), data export/import |

---

## Data Persistence

- **Primary**: `localStorage` key `sj_v3` (auto-saved after every action)
- **Backup/transfer**: Export/Import JSON via Settings → 数据管理
- Avatars stored as base64 data URIs inside the JSON

---

## Players (Hardcoded Defaults)

| ID | Name | Seat |
|---|---|---|
| 0 | BR | 0 |
| 1 | HZB | 1 |
| 2 | JR | 2 |
| 3 | TFH | 3 |
| 4 | WYH | 4 |
| 5 | YY | 5 |

Avatars are embedded as base64 in `index.html`. Names and seat order are editable in Settings and persisted to localStorage.

---

## Deployment

Hosted via **GitHub Pages** from the `main` branch root.
- URL: `https://jimyuhaowu.github.io/shengji/`
- To update: replace `index.html` in the repository root and commit

### Adding to iPhone Home Screen
1. Open the URL in **Safari**
2. Tap Share → "Add to Home Screen"
3. The app runs full-screen offline after the initial load

---

## Known Limitations / Future Ideas

- [ ] Real-time multi-device sync (previously attempted with JSONBin.io, removed)
- [ ] Seat adjacency analysis (does sitting next to someone affect win rate?)
- [ ] Cross-session level progression chart in 生涯 tab
- [ ] Push notifications when it's your turn (requires native app or service worker)
- [ ] Configurable player count (currently hardcoded to 6)
- [ ] Dark/light theme toggle
- [ ] Round undo (currently edit collapses all subsequent rounds)

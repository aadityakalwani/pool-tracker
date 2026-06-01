# Pool Tracker Game

Single-file vanilla web app for scoring a custom 3-player pool game. Zero dependencies. No build step.

## Game Mechanics

| Button | Ball | Points |
|---|---|---|
| Red | Red ball potted | +1 |
| Yellow | Yellow ball potted | +2 |
| Black | Black ball potted | +10 |
| White / Cue | Cue ball potted or off table | −3 |
| Miss | Nothing potted | 0 |

One tap = one turn. Every button press advances to the next player.

## Running Locally

```
cd "/Users/aadityakalwani/Documents/Pool Tracker Game"
python3 -m http.server 8080
```

Open http://localhost:8080 in a browser.

## Deployment

Push to main → Netlify auto-deploys in ~30 seconds.

```
git add index.html && git commit -m "..." && git push
```

## Architecture

Everything lives in `index.html` (HTML + embedded CSS + embedded JS). No framework, no dependencies.

### Screens (CSS class-switching)

1. **Setup** (`screen-setup`) — 3 player name inputs, pre-filled from localStorage. "Start Game" navigates to game screen.
2. **Game** (`screen-game`) — amber banner showing current player, score strip (3 cards), 5 action buttons, sticky footer with Undo + End Game.
3. **History** (`screen-history`) — all-time leaderboard, past games grouped by date.
4. **End Game Modal** (`modal-endgame`) — bottom sheet with final scores and winner, Save & New Game / Save & View History.

### State Variables

```js
const SCRIPT_URL = "";         // paste Apps Script URL here — top of <script> block
let playerNames = [...];       // loaded from localStorage (key: 'pool_player_names')
let scores      = [0, 0, 0];
let currentIdx  = 0;           // index of current player (0/1/2)
let turnCount   = 0;
let undoStack   = [];          // { playerIdx, scoresBefore[] } — in-memory only, cleared on game end
let allGames    = [];          // loaded from Sheets on History screen
```

### Key Functions

| Function | Purpose |
|---|---|
| `startGame()` | Read name inputs, save to localStorage, reset state, show game screen |
| `recordPot(ballType)` | Push undo snapshot, apply delta, advance `currentIdx`, re-render |
| `undoLast()` | Pop undo stack, restore scores and player index |
| `renderGameScreen()` | Update banner, score cards, undo button disabled state |
| `confirmEndGame()` | Populate and open End Game modal |
| `saveGame()` | Fire-and-forget POST to Apps Script (`mode: 'no-cors'`). No-op if SCRIPT_URL empty |
| `saveAndReset()` | Save, toast, back to setup |
| `saveAndViewHistory()` | Save, toast, fetch history after 800ms delay, show history screen |
| `loadAndShowHistory()` | Fetch all games from Sheets, render leaderboard + past games |
| `buildLeaderboard(games)` | Aggregate total scores per player name, sort descending |
| `showToast(msg)` | 2.8s toast notification |

## Google Sheets Backend

Sheet name: `Sessions`

| Column | A | B | C |
|---|---|---|---|
| Field | Date | PlayersJSON | WinnerName |

`PlayersJSON` is a JSON string of the form `[{"name":"Aadi","score":12},{"name":"Sis","score":4}]`. Variable-length player support is why this column exists rather than fixed P1…Pn columns.

### Apps Script Endpoints

All via GET (Apps Script only has `doGet`):

- `?action=list` — returns JSON array of all games, newest first
- `?action=add&data=JSON` — appends a row. Data fields: `date, playersJson, winner`

The full Code.gs is embedded as a comment block at the bottom of `index.html`.

### Deploy Settings

- Execute as: **Me**
- Who has access: **Anyone**

After editing Code.gs: Deploy > Manage Deployments > Edit > New Version > Deploy. Copy new URL if it changed.

## Colour Palette

```css
--bg:           #f7f6f3   warm off-white
--surface:      #ffffff
--border:       #e8e6e0
--text:         #1c1c1e
--muted:        #8e8e93
--accent:       #f59e0b   amber (buttons, active player border)
--ball-red:     #dc2626
--ball-yellow:  #ca8a04
--ball-black:   #1c1c1e
```

## Netlify Setup (first time)

1. `git init` in this directory
2. `git add index.html CLAUDE.md && git commit -m "Initial pool tracker"`
3. Create `aadityakalwani/pool-tracker` on GitHub
4. `git remote add origin <url> && git push -u origin main`
5. Netlify > Add new site > Import from GitHub > select repo > build command: empty, publish dir: `.` > Deploy

## localStorage Keys

| Key | Value |
|---|---|
| `pool_player_names` | JSON array of 3 name strings |

## Notes

- **Name consistency:** Cumulative totals aggregate by exact string match on player name. Casing differences ("aadi" vs "Aadi") create separate leaderboard entries. localStorage defaults minimise this.
- **Fire-and-forget writes:** No retry, no confirmation callback. If a write fails silently, that game's data is lost. Acceptable trade-off for simplicity.
- **Undo is in-memory only:** `undoStack` is cleared when a game ends. No cross-game undo.
- **No input validation:** The app does not enforce pool rules. Any sequence of button presses is valid.

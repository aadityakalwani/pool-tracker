# Pool Tracker Game

Single-file vanilla web app for scoring a custom 3+-player pool game. Zero dependencies. No build step.

## Game Mechanics

| Button | Ball | Points |
|---|---|---|
| Red | Red ball potted | +1 |
| Yellow | Yellow ball potted | +2 |
| Black | Black ball potted | +10 |
| White / Cue | Cue ball potted or off table | −3 |
| Miss | Nothing potted | 0 |
| Custom Pot | Multiple balls in one shot | steppers |

One tap = one turn. Every button press (including Miss and Custom Pot) advances to the next player.

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

GitHub: https://github.com/aadityakalwani/pool-tracker

## Architecture

Everything lives in `index.html` (HTML + embedded CSS + embedded JS). No framework, no dependencies.

### Screens (CSS class-switching)

1. **Setup** (`screen-setup`) — dynamic player name inputs (2–8 players), pre-filled from localStorage. "Start Game" navigates to game screen.
2. **Game** (`screen-game`) — amber banner showing current player, score strip (score + ball breakdown + pts/turn per card), 6 action buttons (Red/Yellow/Black/White/Miss/Custom Pot), sticky footer with Undo + End Game.
3. **History** (`screen-history`) — all-time leaderboard with ball counts + pts/game rate, past games grouped by date.
4. **End Game Modal** (`modal-endgame`) — bottom sheet with sorted scores + ball breakdown, Save & New Game / Save & View History.
5. **Custom Pot Modal** (`modal-custom`) — steppers for Red/Yellow/Black/White counts, live net calculation, Apply button.

### Player Object

```js
// Each player in the players[] array:
{
  name:   string,
  score:  number,  // running total for this game
  red:    number,  // balls potted of each type
  yellow: number,
  black:  number,
  white:  number,  // fouls (cue ball potted / off table)
  turns:  number   // total turns taken
}
```

### State Variables

```js
const SCRIPT_URL = "...";      // Apps Script URL — top of <script> block (already set)
let players      = [];         // array of player objects (see above)
let currentIdx   = 0;
let turnCount    = 0;
let undoStack    = [];         // { playerIdx, snap: deep copy of players[] }
let allGames     = [];         // loaded from Sheets on history screen
let customCounts = { red: 0, yellow: 0, black: 0, white: 0 };  // custom pot modal state
```

### Key Functions

| Function | Purpose |
|---|---|
| `startGame()` | Read name inputs, save to localStorage, init player objects, show game screen |
| `snapshotPlayers()` | Return shallow copy of each player object for undo |
| `recordPot(ballType)` | Snapshot, apply delta + increment ball count + turns, advance idx |
| `undoLast()` | Pop undo stack, restore full players snapshot and currentIdx |
| `renderGameScreen()` | Update banner, score cards (value + breakdown + pts/turn), undo state |
| `openCustomModal()` | Reset counts, set title to current player, show modal |
| `adjustCustom(ball, dir)` | Stepper ±1 on a ball type, refresh net display |
| `applyCustom()` | Snapshot, apply net score + all ball counts, advance turn |
| `confirmEndGame()` | Populate End Game modal: scores sorted desc + ball breakdowns |
| `saveGame()` | Fire-and-forget POST to Apps Script with full player state |
| `saveAndReset()` | Save, toast, back to setup |
| `saveAndViewHistory()` | Save, toast, 800ms delay, load history screen |
| `loadAndShowHistory()` | Fetch games from Sheets, render leaderboard + past games |
| `buildLeaderboard(games)` | Aggregate scores, wins, ball counts across all saved games |
| `showToast(msg)` | 2.8s toast notification |

## Google Sheets Backend

Sheet name: `Sessions` (column header: `PlayerJSON` — the S is missing in the actual sheet but doesn't matter since Apps Script uses column index)

| Column | A | B | C |
|---|---|---|---|
| Field | Date | PlayersJSON | WinnerName |

`PlayersJSON` stores a JSON array per game:
```json
[{ "name": "Aadi", "score": 1, "red": 5, "yellow": 4, "black": 0, "white": 4, "turns": 13 }, ...]
```

Old rows (from before ball counts were added) will have `red/yellow/black/white/turns` missing — the frontend defaults them to 0 via `Number(p.red) || 0`.

### Apps Script Endpoints

- `?action=list` — returns JSON array, newest first
- `?action=add&data=JSON` — appends a row. Fields: `date, playersJson, winner`

Full Code.gs is embedded as a comment block at the bottom of `index.html`.

### Deploy Settings

- Execute as: **Me**
- Who has access: **Anyone**

After editing Code.gs: Deploy > Manage Deployments > Edit > New Version > Deploy.

## Colour Palette

```
--bg:          #f7f6f3   warm off-white
--surface:     #ffffff
--border:      #e8e6e0
--text:        #1c1c1e
--muted:       #8e8e93
--accent:      #f59e0b   amber (buttons, active player border)
--ball-red:    #dc2626
--ball-yellow: #ca8a04
--ball-black:  #1c1c1e
```

## Netlify Setup (already done)

GitHub → Netlify auto-deploy on push to main. No build command, publish dir `.`.

## localStorage Keys

| Key | Value |
|---|---|
| `pool_player_names` | JSON array of player name strings |

## Notes

- **Name consistency:** Leaderboard aggregates by exact string. "Aadi" ≠ "aadi". localStorage defaults minimise typos. The test game used "kishu" — this will appear as a separate leaderboard entry from "Kunaal" until the test row is deleted from the Sheet.
- **Fire-and-forget writes:** No retry. If a write fails silently, that game is lost. Acceptable trade-off.
- **Undo:** Full player snapshot (scores + ball counts). In-memory only, cleared when game ends.
- **Custom Pot:** For multi-ball shots (e.g. two yellows + one white on the same hit). Counts as one turn.
- **Ball counts in old data:** Rows saved before this feature was added will show 0 for all ball types. Scores are unaffected.

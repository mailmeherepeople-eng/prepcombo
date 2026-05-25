# PrepCombo — Claude Code Handoff

Single-file HTML/JS game for PrepLadder. Medical education card-matching game.
Runs inside PrepLadder's Android/iOS WebView. No framework, no build step, no CDN.

## File
`index.html` — ~1170 lines. All CSS, HTML, JS inline. This is the only file.

## How to test
Open `index.html` directly in a browser. No server needed.
To test localStorage persistence (high score, dark mode, onboarding): run in a browser,
not via file:// if localStorage is blocked — use `python3 -m http.server` if needed.

To skip training and jump straight to the game for testing, open browser console and run:
```js
localStorage.setItem('prepcombo_trained_v1','1'); location.reload();
```

To reset high score for testing new record flow:
```js
localStorage.removeItem('prepcombo_hs'); location.reload();
```

## Architecture

**No frameworks. No imports. No build.**
Pure vanilla JS/CSS/HTML. All game state is in module-level `let` variables.
DO NOT introduce any npm dependencies, CDN links, or build tools.

**WebView bridge (do not break):**
```js
window.onPrepComboEnd(result)
// result = { score, rank, plays: gameLog[], timestamp }
// gameLog entry: { round, bucket, cards, pts, mult, speedBonus, correct, hod, correctCount? }
```
This fires at the end of every game. PrepLadder's native app listens to it.
Any new analytics hooks should follow the same pattern (e.g. `window.onPrepComboRound`).

**localStorage keys in use:**
- `prepcombo_trained_v1` — onboarding completion flag
- `prepcombo_dark` — dark mode preference ('1' or '0')
- `prepcombo_hs` — all-time high score (integer as string)

**CSS variables (respect these — dark mode depends on them):**
```
--bg, --surface, --surface2, --card-border
--accent, --gold, --text-primary, --text-secondary
--card-face, --card-hint, --card-divider
```
Never hardcode colors inside a rule that needs to work in both light and dark.
Use `var(--card-face)` for card backgrounds, not `#f5f0e8`.

**Content rules:**
- No em dashes (`—`) anywhere in player-facing text. Use hyphens or nothing.
- Correct messages must mention PrepLadder (marketing requirement).
- Wrong messages use medical humor. See `wrongMessages[]` array.

## Game flow
Splash → Start screen → (Onboarding → Training → ) → Game → Result

Constants: `ROUNDS=3`, `TURNS_PER_ROUND=3` (implied by advanceTurn logic), `ROUND_TIME=120`

## Key functions to know

| Function | What it does |
|---|---|
| `startGame()` | Generates all rounds, initializes state, calls `loadRound()` |
| `loadRound()` | Renders cards + buckets for current round. Adds entrance animation. |
| `advanceTurn()` | Increments turn, triggers round splash or endGame |
| `dropInBucket(bucketId, bucketEl)` | Core submit logic. Handles correct, wrong, HOD |
| `triggerHouseStatus(tag, text, isSnoop)` | HOD modal + GIF + confetti |
| `endGame()` | Shows result screen, HS logic, count-up animation, WebView bridge |
| `launchConfetti(gold=false)` | Gold=true for new record (more pieces, all gold/white) |
| `animateCount(el, from, to, ms)` | Ease-out count-up for score display |
| `showRecordBanner()` | Injects animated "NEW RECORD" overlay div, auto-removes after 3s |
| `pickRoundTags()` | Selects 6 tags per round. Filter: `tagMap[t].length >= 3` |
| `buildRound(tags)` | For each tag, picks 3 cards at random (handles pools > 3) |
| `buildTrainingBuckets()` | Shows relevant subjects only (fixed — was showing all 22) |

## What was just shipped (Track A + B + partial C)

**Track A — bug fixes:**
- tagMap filter changed from `=== 3` to `>= 3` (Bacteria tag now draws correctly)
- buildRound now randomly samples 3 from pools > 3
- Score rank thresholds recalibrated (Dean of Medicine now >2800, not >6000)
- Training buckets now show only relevant subjects + distractors to 12 total
- Turn counter now shows "1/3", "2/3", "3/3" format
- HOD GIF has onerror fallback (🏆 or 🌿 emoji)

**Track B — visual polish:**
- Dark mode (body.dark class, moon/sun toggle, localStorage persisted)
- Card entrance animation with stagger on round load
- Round splash has progress dots (done/active/pending)
- HOD modal redesigned: tag badge pill, treasure in frosted box, GIF fallback
- History items: hi-wrong (red border), hi-hod (gold border)

**Track C — partial (just shipped):**
- High score persistence (`prepcombo_hs` localStorage key)
- Score count-up animation on result screen
- New record: gold confetti, pulsing rank box border, "NEW RECORD" banner
- When new record + previous best exists: count animates to prev best, pauses, then bursts through
- Near-miss feedback on wrong answers:
  - 2/3 correct → message says which card was the odd one out (e.g. "2/3 were right. Troponin was the odd one out.")
  - 0/3 or 1/3 → normal funny wrong message
  - ALL wrong answers now show near-miss line in history sidebar item

---

## Track C — remaining work (implement these next)

### 1. Daily Challenge mode
**Goal:** Same game every day for everyone. Biggest retention feature.

**How:**
- Add a "Daily Challenge" button on the start screen alongside "Start Rotation"
- Use date as RNG seed for `pickRoundTags()`:
  ```js
  function seededRand(seed){
      let s=seed%2147483647;
      return ()=>{s=s*16807%2147483647;return(s-1)/2147483646;};
  }
  const today=new Date().toISOString().slice(0,10); // "2026-05-25"
  const seed=today.split('').reduce((a,c)=>a+c.charCodeAt(0),0);
  ```
- Store completion in `localStorage.setItem('prepcombo_daily_'+today, score)`
- If already played today: show "Come back tomorrow. Today's score: X pts"
- Result screen for daily: different h1 ("Daily Rotation Complete"), show date, share text = "PrepCombo Daily — [date] — X pts"
- No backend required. Deterministic seed = everyone on same date gets identical game.

**localStorage keys to add:**
- `prepcombo_daily_YYYY-MM-DD` — stores score as string for that date

---

### 2. Escalating HOD chain
**Goal:** Consecutive HOD turns within a game get increasingly dramatic celebrations.

**How:**
- Track `hodStreak` (separate from `streak`) — increments on HOD, resets on non-HOD (correct or wrong)
- HOD chain 1: normal (current behavior)
- HOD chain 2: title changes to "Double Department!" + slightly more confetti
- HOD chain 3+: title "Full Professorship!" + gold confetti + extra haptic pattern

**State to add:**
```js
let hodStreak = 0; // reset in advanceTurn or on wrong/non-HOD correct
```

**Where to modify:** `triggerHouseStatus()` — pass hodStreak, vary title and confetti.
Also reset `hodStreak` in the wrong path of `dropInBucket`.

---

### 3. Round 3 Final Exam Mode
**Goal:** Round 3 locks card hints (no long-press flip on turns 1 and 2). Creates tension escalation.

**How:**
- In `makeCard()`, the longPress handler is set via the passed `longPressFn` parameter
- In `loadRound()`, pass a no-op for longPressFn when `currentRound === 2 && currentTurn < 3`:
  ```js
  const lpFn = (currentRound===2 && currentTurn<3)
      ? null
      : (el,data)=>longPressFlipCard(el,data);
  currentRoundCards.forEach(data=>{
      makeCard(data,pool,(el,data)=>tapSelectCard(el,data), lpFn);
  });
  ```
- Show a brief message when Round 3 loads: "Final Exam Mode. No hints until the last turn."
- Optionally: show a small 🔒 badge on unflippable cards

---

### 4. Skip Turn mechanic
**Goal:** One skip per game. Player returns 3 selected cards to pool without submitting.

**How:**
- Add `skipsRemaining = 1` to game state, reset on `startGame()`
- Show a "Skip Turn" button near the submit hint area. Only visible when 3 cards are selected AND `skipsRemaining > 0`
- On tap: deselect all cards, advance turn WITHOUT logging a play, decrement `skipsRemaining`
- The skipped turn still advances (turn 1 → turn 2), cards return to pool visible
- Show a "Skip used." message after. Disable button for rest of game.

**UI placement:** Below `#submit-hint`, above `#timer-bar-wrap`. Only show when relevant.

---

### 5. Content pass — fill dead tags
**Goal:** 3 tags currently have < 3 cards and never draw. Add cards to make them playable.

**Current dead tags:**
- `Nutrition` — has Kwashiorkor, Pellagra. **Add:** Scurvy (Vitamin C deficiency, Dermatology + Medicine)
- `Sepsis` — has only Septic Shock. **Add:** Bacteraemia, Organ Failure (Pathology + Medicine each)
- `DNA/genetics` — has only DNA Repair. **Add:** PCR (Biochemistry + Pathology), CRISPR (Biochemistry + Genetics)

**Card data format (copy from existing cards for structure):**
```js
{name:'Scurvy', emoji:'🍋', buckets:['Dermatology','Medicine'], tags:['Nutrition'],
 hint:'Vitamin C deficiency. Perifollicular hemorrhage, corkscrew hairs.',
 hintLast:'Vitamin C deficiency — Dermatology and Medicine crossover.'}
```

---

### 6. WebView analytics hook
**Goal:** Let PrepLadder track per-round data, not just end-of-game.

**How:** Fire `window.onPrepComboRound` at end of each round (before round splash).
Add to `advanceTurn()` when `currentTurn > 3`:
```js
if(typeof window.onPrepComboRound==='function'){
    window.onPrepComboRound({
        round:currentRound,
        plays:gameLog.filter(p=>p.round===currentRound),
        roundScore: gameLog.filter(p=>p.round===currentRound).reduce((a,p)=>a+p.pts,0)
    });
}
```

---

## Design constraints (non-negotiable)

1. No em dashes in player-facing text
2. No external dependencies (no CDN, no npm)
3. WebView bridge format must stay intact
4. All CSS must use CSS variables for colors (dark mode compatibility)
5. Correct messages must mention PrepLadder
6. `makeCard()` and the long-press flip are battle-tested on mobile — do not refactor the touch event handling
7. The `.card-inner` perspective transform + `backface-visibility:hidden` pattern is intentional — transforming the parent wrapper breaks it

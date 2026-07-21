# The Music Quiz — project brief

A buzzer/quiz app for home quiz nights. **Single file: `index.html`** — plain HTML/CSS/JS, no build step, no framework. One page that renders different "views" from URL params.

- **Live:** https://jj-wcw.github.io/music-quiz/ · **Repo:** https://github.com/JJ-WCW/music-quiz (public)
- **Deploy = `git push` to `main`.** GitHub Pages rebuilds in ~40s. If a build hangs (>3 min "building"), re-trigger: `gh api -X POST repos/JJ-WCW/music-quiz/pages/builds`.
- **`./quiz-offline`** takes the public link down (404s) between quiz nights; **`./quiz-online`** brings it back. Pages can't be password-protected on this plan — see "Privacy" below.

## Views and URLs

`?view=home|join|player|display|admin` and `?bucket=<GAME_CODE>`.

- **admin** — the host's laptop dashboard (never cast). Password-gated.
- **display** — the projector tab (cast this). Lobby/QR, questions, banners, standings, podium.
- **player** — a phone: buzzer, answer buttons, text input, wager betting.
- **join** — sign-up (name, avatar, buzzer sound). **home** — landing.

## Backend

Firebase Realtime Database over plain REST (no SDK, no accounts):
`https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app`

Everything for one game lives under `/games/<code>/`:

| Key | What |
|---|---|
| `qb-state` | the live game state (one object; see `defaultState()`) |
| `qb-players` | array of `{id,name,avatar,score,buzzSound,...}` |
| `qb-rounds` | the quiz library (see below) |
| `qb-admin` | sha256 hex of the admin password |
| `qb-buzz-<epoch>/<playerId>` | one buzz |
| `qb-answer-<epoch>/<playerId>` | one question answer |
| `qb-ante-<epoch>/<playerId>` | one wager bet |

`kvGet` returns raw text → `JSON.parse`; `kvSet` PUTs a JSON string (Firebase parses the body and stores structured data). Game-code paths are **case-sensitive** — the app lowercases codes.

One path is **not** a game: `/games/qhistory/asked.json` is the cross-quiz **question history** — every question/song ever used, appended by the quiz-generation recipe (merge `PATCH`, never `PUT`) and checked before generating so quiz nights never repeat material. Schema and workflow live in `QUIZ-RECIPE.md`. The app itself never reads or writes it.

## The library: rounds are groups of questions

`qb-rounds` = `[...groups, ...suddenDeaths]`.

- **Group (a "round")**: `{id, type:'group', name:'General Knowledge', questions:[...]}`
- **Questions** (inside a group, any mix): `type:'song'` (layered music buzzer), `type:'question'` (4-way multiple choice), `type:'text'` (typed answer), `type:'slider'` (slide-to-guess a number)
- **Sudden death**: `type:'sudden'`, top level, **after** all groups. Sits outside the running order (no "Question X of Y", never counts as the final round).
- Legacy flat libraries are auto-folded into a single "Round 1" group by `upgradeLibrary()`.

**Full schema + how to bulk-generate a quiz: see `QUIZ-RECIPE.md`.** A fresh session can read that file alone and `curl` a whole quiz into a game code.

## Core mechanics

- **Song rounds**: instrument layers revealed one at a time, fewer layers = more points. First buzz locks everyone out. **One guess per song** (wrong = out for that song, buzzers reopen for everyone else). 15s to answer after buzzing.
- **Question start**: multiple-choice/text/slider rounds open with a **3-2-1 countdown** on the projector (`state.countdown`), so the question doesn't jump-scare the room. The host tab owns the clock: `loadRoundToPlay`/`closeBetting` set `countdown:true` with no `answerEpoch`, and `adminTick`'s `startCountedQuestion()` mints the epoch and `stampBuzzersOpened()` when it ends — so nothing is answerable or server-stamped until "GO". Song and sudden-death rounds have no countdown (they open the buzzers immediately).
- **Question/text rounds**: everyone answers on their phone against a per-round timer (`seconds`, default 15 / 25 for typing). No lockout. Fastest correct gets a bonus. **Multiple-choice and slider questions auto-end ~2s after every player has answered** (`adminAutoEnd`, fired by the host tab's `adminTick` — single writer); text rounds never auto-end because typed answers may be revised until the whistle. **Text rounds go to a host review screen** — fuzzy matching (case/spacing/punctuation/accents ignored) pre-ticks near-misses, host approves with ✓/✗, nothing scores until "Confirm & score it".
- **Slider rounds** (`type:'slider'`): host sets min/max/answer; players drag on their phone. **Nearest guess wins** and ties genuinely share. Optional `sliderBand` — everyone within ±band also scores (leave 0 for cut-throat "guess the year"; set one for huge ranges like PS2 sales). A **bullseye** (`dist <= step/2`, i.e. the closest notch the slider can reach) takes the `bonus`; with a step of 1 that collapses to an exact hit. Scores itself — no review screen. The kitty still goes to one person: closest, fastest breaking a dead heat.
  - **The step is derived, not typed**: `niceStep()` = range/100 snapped to a 1/2/5×10ⁿ ladder, so 2m-10m slides in 100,000s and 1990-2025 slides in 1s. `sliderStepFor()` resolves it **once at load time** into `state.sliderStep`, so every view agrees; don't recompute it per view. A `sliderStep` override on the round wins.
  - **Number formatting is per-question**: `groupNums(sliderMax)` decides thousands separators, because "160,000,000" needs them and "2,007" is just wrong. Every `fmtNum`/`fmtGuess` call must pass it.
- **Wagers**: set `isWager` + `wagerClue` on **any** question type. Projector shows the clue first; players bet up to `MAX_BET` (100) into a kitty, **scores may go negative**. Players who don't fancy it tap **No bet**, which writes a `{amount:0, pass:true}` ante — that's what makes `antesDecided()` reach the player count so the host knows to stop waiting. Host closes betting, question plays normally, **the best-placed answerer who actually staked takes the whole kitty** — `settleKitty()`/`markCorrect()` filter to `stakedIn()`, so a non-bettor who was closer or faster still wins the normal points but **can't touch the pot** (it rolls over instead). If **exactly one player bet and that player wins**, the house matches their stake on top of the pot (`loneBetBonus()` — otherwise a solo bettor just wins their own points back, which made betting alone pointless); `state.betMatched` carries the matched amount to the reveal messaging. Betting is optional; everyone keeps their buzzer.
- **The kitty rolls over**: nobody wins it → `rollKittyOver()` banks the pot in `state.carryKitty` instead of burning it, and it survives round loads (like `quizTitle`) until someone finally wins, growing each time. `kittyTotal(s)` = this round's stakes **+** the carry — use it everywhere rather than summing `bets` by hand. A win zeroes `carryKitty`; `defaultState()` means a reset clears it too.
- **Ties**: when the last question of the **last** group reveals with the top level, admin shows a gold tie panel with one-click "Run sudden death". Loading a sudden death during a tie snapshots the tied leaders into `sdEligible` — **only their buzzers work**. The podium button soft-confirms over a tie (never hard-blocks).
- **Podium**: staged 3rd/2nd/1st reveal with a synthesized track. Host-triggered.

## Non-obvious invariants — don't break these

1. **Race-free writes.** Buzzes/answers/bets each write **only their own key** under an epoch, stamped with Firebase's server clock (`{'.sv':'timestamp'}`); the earliest timestamp wins. **Never** read-modify-write a shared array from a player device. `qb-players` (scores) is written **only by the host tab** — single writer.
2. **Server time only.** Reaction/answer times compare `ts` to `state.buzzersOpenedAt`, both server-stamped. Never trust phone clocks.
3. **Polls rebuild the DOM** (display 900ms, player 800ms, admin 1200ms) — this has caused several bugs:
   - Don't rebuild anything with an `<img>` or an input every poll. Rebuilding restarts the image load forever (so it never appears) and throws away a half-dragged slider or half-typed answer.
   - **Display**: live questions **render once** and only patch the answered-chips (`data-qepoch` + `#qb-answered-row`).
   - **Player**: every render fingerprints everything that decides the screen's *shape* (`prevPlayerShape`). Same shape → patch the live numbers by id (`#qb-p-score`, `#qb-p-kitty`, `#qb-p-pot`, `#qb-p-decided`, `#qb-p-answered`, `#qb-p-scoreboard`) and return. **Anything new that changes the phone's structure must go in that fingerprint**, or it'll never appear.
   - The player poll deliberately has **no `document.activeElement` guard**: the fingerprint already protects inputs, and skipping the poll on focus would strand a phone (iOS doesn't move focus when you tap a button) on a dead question forever. Don't "fix" this by re-adding the guard.
   - Admin preserves per-panel scroll across repaints via `data-scrollkey`.
   - Don't poll at all when there's no `bucket` (the setup screen) — it stole focus.
   - Admin form inputs that feed a live-recalculated hint (the slider's step/notches note) bind `onchange`, **not** `oninput` — re-rendering mid-keystroke steals focus. Never give such an input a `data-action`; `bindActions` wires those to `onclick`, so clicking into the field would re-render it out from under the cursor.
4. **Countdowns**: bake the *current* remaining value into each render; never reset the timer on a transient failed fetch (only a new epoch/reveal clears it).
5. **Stale renders**: every polled view takes a sequence ticket (`displaySeq`/`playerSeq`/`adminSeq`) and abandons itself if superseded.
6. **Audio**: Web Audio is synthesized (no files). `playFx` queues on `AudioContext.resume()` so the first tap is audible; a keepalive resumes it if the browser suspends it. iOS silent switch still mutes phones — the lobby tells players to unmute. Locking in any answer (MC/text/slider) plays that player's own buzzer sound (`playBuzzFor`), *before* the network write so it feels instant; reveal plays a correct ding / wrong jingle on each phone for their own result only — room-wide noise belongs to the display.

## Working conventions

- **Always syntax-check before shipping**: extract the inline `<script>` and run it through `new Function(src)` (JSC via `osascript -l JavaScript`).
- **Test against the real database** with a throwaway game code, then **delete the test game** (`curl -X DELETE .../games/<code>.json`). Screenshot visual changes.
- Admin is desktop-only by design (fixed-height dashboard, columns scroll individually). Projector/phones get the vibrant themed look; admin stays plain.
- Visual cues (played ✓, ▶ NOW) must **never disable** a button — the host may always redo anything.
- Confirm before destructive/irreversible host actions (loading over a live round, removing a player, resetting).

## Privacy / security posture

The Pages site is public (GitHub can't password-protect it on this plan; private access control is Enterprise-only). The Firebase DB is **open read/write under `/games/<code>`** by design — you can't *list* games, but anyone with the DB URL + a specific code can read/write that game, bypassing the client-side-only admin password. Fine for a home quiz. Use random auto-generated codes (not memorable ones) if you want the questions kept secret until the night. `./quiz-offline` hides the *app*, not the *data*.

## Known limitations

- Polling (~1s), not websockets — fine for a living room, not for split-second precision.
- Players who clear their phone's storage lose their identity (localStorage resume + "that's me" claim covers refreshes).
- Firebase free tier is far beyond what a living room needs.

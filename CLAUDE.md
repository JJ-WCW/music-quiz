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

## The library: rounds are groups of questions

`qb-rounds` = `[...groups, ...suddenDeaths]`.

- **Group (a "round")**: `{id, type:'group', name:'General Knowledge', questions:[...]}`
- **Questions** (inside a group, any mix): `type:'song'` (layered music buzzer), `type:'question'` (4-way multiple choice), `type:'text'` (typed answer)
- **Sudden death**: `type:'sudden'`, top level, **after** all groups. Sits outside the running order (no "Question X of Y", never counts as the final round).
- Legacy flat libraries are auto-folded into a single "Round 1" group by `upgradeLibrary()`.

**Full schema + how to bulk-generate a quiz: see `QUIZ-RECIPE.md`.** A fresh session can read that file alone and `curl` a whole quiz into a game code.

## Core mechanics

- **Song rounds**: instrument layers revealed one at a time, fewer layers = more points. First buzz locks everyone out. **One guess per song** (wrong = out for that song, buzzers reopen for everyone else). 15s to answer after buzzing.
- **Question/text rounds**: everyone answers on their phone against a per-round timer (`seconds`, default 15 / 25 for typing). No lockout. Fastest correct gets a bonus. **Text rounds go to a host review screen** — fuzzy matching (case/spacing/punctuation/accents ignored) pre-ticks near-misses, host approves with ✓/✗, nothing scores until "Confirm & score it".
- **Wagers**: set `isWager` + `wagerClue` on **any** question type. Projector shows the clue first; players bet up to `MAX_BET` (10) into a kitty, **scores may go negative**. Host closes betting, question plays normally, **the winner takes the whole kitty**. Betting is optional; everyone keeps their buzzer.
- **Ties**: when the last question of the **last** group reveals with the top level, admin shows a gold tie panel with one-click "Run sudden death". Loading a sudden death during a tie snapshots the tied leaders into `sdEligible` — **only their buzzers work**. The podium button soft-confirms over a tie (never hard-blocks).
- **Podium**: staged 3rd/2nd/1st reveal with a synthesized track. Host-triggered.

## Non-obvious invariants — don't break these

1. **Race-free writes.** Buzzes/answers/bets each write **only their own key** under an epoch, stamped with Firebase's server clock (`{'.sv':'timestamp'}`); the earliest timestamp wins. **Never** read-modify-write a shared array from a player device. `qb-players` (scores) is written **only by the host tab** — single writer.
2. **Server time only.** Reaction/answer times compare `ts` to `state.buzzersOpenedAt`, both server-stamped. Never trust phone clocks.
3. **Polls rebuild the DOM** (display 900ms, player 800ms, admin 1200ms) — this has caused several bugs:
   - Don't rebuild anything with an `<img>` or an input every poll. Live questions **render once** and only patch the answered-chips (`data-qepoch` + `#qb-answered-row`). Rebuilding restarted the image load forever and jolted the layout.
   - Guard repaints while the user types (`document.activeElement` checks on bet/text inputs).
   - Admin preserves per-panel scroll across repaints via `data-scrollkey`.
   - Don't poll at all when there's no `bucket` (the setup screen) — it stole focus.
4. **Countdowns**: bake the *current* remaining value into each render; never reset the timer on a transient failed fetch (only a new epoch/reveal clears it).
5. **Stale renders**: every polled view takes a sequence ticket (`displaySeq`/`playerSeq`/`adminSeq`) and abandons itself if superseded.
6. **Audio**: Web Audio is synthesized (no files). `playFx` queues on `AudioContext.resume()` so the first tap is audible; a keepalive resumes it if the browser suspends it. iOS silent switch still mutes phones — the lobby tells players to unmute.

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

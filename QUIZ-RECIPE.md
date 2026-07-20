# Auto-building a quiz (recipe for Claude)

**If you are Claude and the user asked you to generate/build/upload a quiz, read this whole file, then do it.** You do not need the rest of the codebase or any long chat history — everything you need is here. This is designed so a **fresh, cheap session** can build a full quiz in one shot.

## The one-line idea

The entire quiz library is a single JSON document stored in the user's Firebase database at:

```
https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/<GAME_CODE>/qb-rounds.json
```

You generate the questions, shape them into the JSON below, and `PUT` that JSON to the URL above. The app reads it live. No login/token is needed (the database is open write under `/games/<code>` by design).

## Before you start

1. The user must have **started a game in the app first** (opens `?view=admin`, sets a password, gets a **game code**). Ask for the game code if they haven't given one. You only ever write the questions; the game/password already exist.
2. To also set the **quiz title** (optional), `PUT` a JSON string to `/games/<CODE>/qb-state/quizTitle.json` — e.g. body `"Music + Trivia"`.
3. **Writing `qb-rounds` replaces the whole library** for that code. If they want to *add* to an existing quiz, first `GET` the current `qb-rounds.json`, merge, then `PUT` the whole thing back.

## Never repeat a question from a previous quiz night

The database keeps a history of every question ever used, at `/games/qhistory/asked.json` (that's a reserved pseudo-game-code, not a real game — never write rounds to it). **This is a hard requirement, not a nice-to-have:** repeated questions have happened before and the players notice immediately.

**BEFORE writing any questions**, pull the history:

```bash
curl -s "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/qhistory/asked.json" \
  | python3 -c "import sys,json; d=json.load(sys.stdin) or {}; [print(f\"- {v.get('q')}  →  {v.get('a')}\") for v in d.values()]"
```

Read the list and **avoid semantic repeats, not just exact ones** — "What year did the first iPhone launch?" and "When did Apple release the original iPhone?" are the same question; a song already used in a song round shouldn't come back as a trivia answer either. If a topic overlaps, find a genuinely different fact about it.

**AFTER a verified upload**, append everything you used (every question, text, slider, sudden death, and song — one entry per item) with a merge `PATCH` (never `PUT` — a `PUT` would wipe the history):

```bash
curl -s -X PATCH \
  "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/qhistory/asked.json" \
  --data-binary @history-append.json -w "\nHTTP %{http_code}\n"
```

where `history-append.json` maps a slug (lowercase letters/digits of the question **plus the answer** — the answer matters because picture rounds reuse wording like "Who is this?" — ≤80 chars) to one entry each:

```jsonc
{
  "whatisthecapitalofaustraliacanberra": { "q": "What is the capital of Australia?", "a": "Canberra", "t": "question", "quiz": "<GAME_CODE>", "date": "2026-07-19" },
  "songdontstopmenowqueen":              { "q": "SONG: Don't Stop Me Now — Queen", "a": "", "t": "song", "quiz": "<GAME_CODE>", "date": "2026-07-19" }
}
```

## House style: this is a UK quiz

The players are in the **UK**. Write for them:

- **British spellings** everywhere ("colour", "theatre", "aluminium"), £ not $, metric-first (with imperial only where Brits genuinely use it — miles, pints).
- **UK framing for ambiguous questions**: "the charts" means the UK charts, "the top flight" means the Premier League, "the government" means Westminster. Football means ⚽.
- **No obscure Americana** — nothing that assumes US schooling or US-only culture (state capitals, NFL/NBA/MLB trivia, US TV networks, "grade school" framing). Globally famous American things are fine: US presidents, Hollywood, NASA, global pop culture.
- Pop culture (films, TV, music) is global anyway — no need to force Britishness onto it, just don't lean American where a neutral phrasing exists.
- UK-flavoured material is actively good: British music, British TV and sitcoms, UK geography, royal history — it plays well in the room.

## The JSON shape

`qb-rounds` is a JSON **array** containing, in this order: all the **round groups** first, then any **sudden-death** tie-breakers.

> **Default:** Always add **2 sudden-death tie-breakers** at the end of every quiz automatically, even if the user doesn't mention them — unless the user explicitly says they don't want any. Make them general-knowledge questions with a short, unambiguous answer (a year, number, or single name).

```jsonc
[
  {
    "id": "g1",                 // any unique string, unique across the WHOLE file
    "type": "group",            // a "round" = a named group of questions
    "name": "Geography",        // shown on the projector: "Round 1 of 3 · Geography"
    "questions": [ /* question objects, any mix of the types below */ ]
  },
  // ...more groups...
  { /* sudden-death objects go here, after all groups */ }
]
```

### Question types (these go inside a group's `questions` array)

> **Standard points** (use these unless the user asks for something else): multiple choice = **3**, typed answers = **6** (typing is harder than tapping), sliders = 3. `bonus` — the fastest-finger bonus for the quickest correct answer (on a slider it's the bullseye bonus instead) — is **2 on every question type**.

**Multiple choice** — everyone taps one of four coloured answers on their phone:
```jsonc
{
  "id": "q1",                   // unique string
  "type": "question",
  "label": "Question 1",        // keep as "Question N" within its round
  "question": "What is the capital of Australia?",
  "answers": ["Canberra", "Sydney", "Melbourne", "Perth"],  // EXACTLY 4 strings
  "correctIndex": 0,            // 0-3 — MUST vary from question to question, see below
  "points": 3,                  // standard for multiple choice: 3
  "bonus": 2,                   // fastest correct answer bonus — standard: 2
  "seconds": 15,                // answer time limit (number)
  "imageUrl": ""                // OPTIONAL direct image URL shown as a clue; omit or "" for none
}
```

> **Randomise `correctIndex`.** The four answers show as coloured buttons (0=red, 1=blue, 2=yellow, 3=green) — if you write every question with the right answer first, the right answer is *always red* and players notice within two questions. Spread the correct answers roughly evenly across all four positions over each round, and never put the correct answer in the same position more than twice in a row. Easiest method: write each question with the correct answer first, then rotate/shuffle the array and set `correctIndex` to where it landed. The verify step below checks this.

> **Wagers work on ANY question type** (multiple choice, text, slider, or song — not sudden death). Add these three fields to any question to make it a betting round: `"isWager": true, "wagerClue": "a teasing hint shown BEFORE the question is revealed", "wagerImage": ""` (optional picture). Players bet points into a kitty off the clue alone, then the question plays as normal and **whoever wins it takes the whole kitty** (a lone bettor who wins gets their stake doubled by the house, so solo bets still pay). If nobody wins it, the kitty **rolls over into the next wager round** and keeps growing. Use sparingly — one or two per quiz — and **vary which question type carries the wager**: if a quiz has two wagers, make them two *different* types (e.g. one multiple choice, one slider), and across quiz nights don't default to the same type every time.

**Type-the-answer** — everyone types a free-text answer; the host approves near-misses:
```jsonc
{
  "id": "q2",
  "type": "text",
  "label": "Question 2",
  "question": "Which ocean is the largest?",
  "textAnswer": "Pacific",      // the accepted answer (matching ignores case/spaces/punctuation)
  "points": 6,                  // standard for typed answers: 6 (typing is harder)
  "bonus": 2,                   // fastest correct answer bonus — standard: 2
  "seconds": 20,                // typing needs longer; 20-30 is sensible
  "imageUrl": ""                // optional
}
```

**Slider range** — everyone drags to a number on their phone; closest guess wins. Ideal for "how many/how much/what year" questions where nobody knows the exact figure:
```jsonc
{
  "id": "q4",
  "type": "slider",
  "label": "Question 3",
  "question": "How many PlayStation 2 units have been sold worldwide?",
  "sliderMin": 20000000,        // the ends of the slider — the answer MUST sit between them
  "sliderMax": 200000000,
  "sliderAnswer": 160000000,    // the true figure
  "sliderStep": 0,              // 0 = auto (recommended): derived from the range, ~50-200 notches
  "sliderBand": 5000000,        // 0 = pure nearest-wins; otherwise EVERYONE within ±this also scores
  "sliderUnit": "units",        // optional, shown after the number ("", "%", "years", "units")
  "points": 3,
  "bonus": 2,                   // extra points for a bullseye (the closest notch to the answer)
  "seconds": 25,
  "imageUrl": ""                // optional
}
```

> **Picking `sliderBand`** — this is the one judgement call. Use `0` when the range is small and people can realistically be exact (a year, an age, a percentage): closest wins outright and it stays cut-throat. Use a band of roughly **2-5% of the range** when the range is huge (millions), where nobody will ever be near and "closest wins" would feel arbitrary — a band lets several people score.

> **Spread the answers across the ranges.** Players learn the meta fast: if every answer sits near the middle of its slider, parking the thumb at 50% becomes the winning strategy. For each slider compute the position fraction `(sliderAnswer − sliderMin) / (sliderMax − sliderMin)` and **vary it deliberately across the quiz** — some answers down around 0.1–0.3, some up at 0.7–0.9, a few in the middle. Aim for a roughly even spread over ~0.1–0.9 with no clustering (and never outside that, or the answer hugs an endpoint and the slider feels broken). The easiest lever: pick the range *after* the answer, and stretch it asymmetrically — for a 1994 answer, `1950–2010` puts it at 0.73 while `1975–2015` puts it at 0.48; both feel natural to players. The verify step below prints the fractions — fix any clustering before calling it done.

**Song round** — the layered-music buzzer round the app was built around (only use if the user wants music rounds; they play the audio themselves):
```jsonc
{
  "id": "q3",
  "type": "song",
  "label": "Question 1",
  "songTitle": "Don't Stop Me Now",
  "artist": "Queen",
  "stages": [                   // instrument layers, hardest (fewest layers) first = most points
    { "label": "1 Track Layer", "points": 5 },
    { "label": "2 Track Layers", "points": 4 },
    { "label": "3 Track Layers", "points": 3 }
  ],
  "isWager": false,             // wager fields work here too (see the note above)
  "wagerClue": "",
  "wagerImage": ""
}
```

### Sudden-death tie-breakers (go AFTER all groups, at the top level of the array)
```jsonc
{
  "id": "sd1",
  "type": "sudden",
  "label": "Sudden death 1",
  "question": "In what year did the first iPhone launch?",
  "answer": "2007",             // only the host sees this
  "points": 2
}
```

## Hard rules (get these wrong and the app breaks)

- Every `id` must be **unique across the entire file** (groups, questions, sudden deaths).
- `answers` for a `question` must be **exactly 4** strings; `correctIndex` in **0-3**.
- **Spread `correctIndex` across all four positions** — never mostly one value (see the note under Multiple choice).
- **Check the question history first and never repeat** a previously used question, answer, or song (see "Never repeat a question").
- If there's more than one wager, **use different question types for them**.
- `points`, `bonus`, `seconds`, `correctIndex` must be **numbers**, not strings.
- Use the **standard points** unless told otherwise: `question` = 3, `text` = 6, `slider` = 3, `bonus` = 2 on everything.
- For a `slider`, every `slider*` field must be a **number** (not a string, no commas: `20000000`, never `"20,000,000"`), and `sliderMin < sliderAnswer < sliderMax` — an answer outside the range is unreachable and nobody can win.
- **Vary where slider answers sit in their ranges** — position fractions spread over ~0.1–0.9, not clustered around the middle (see "Spread the answers across the ranges").
- Groups come **first** in the array, sudden deaths **last**.
- Always include **2 sudden-death tie-breakers** by default (see the note under "The JSON shape") unless the user opts out.
- Don't invent other `type` values. Valid: `group`, `question`, `text`, `slider`, `song`, `sudden`.
- Write **real, correct** answers. Double-check facts. For `text` answers, pick a single clear accepted spelling (the host can approve variants live).

## Upload command

Write the JSON to a file, then PUT it (use `--data-binary` so nothing gets mangled):

```bash
curl -X PUT \
  "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/<GAME_CODE>/qb-rounds.json" \
  --data-binary @quiz.json \
  -w "\nHTTP %{http_code}\n"
```

Optional title:
```bash
curl -X PUT \
  "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/<GAME_CODE>/qb-state/quizTitle.json" \
  --data-binary '"Music + Trivia"'
```

## Verify before telling the user you're done

Read it back and sanity-check counts, the correct-answer spread, and the wager mix:
```bash
curl -s "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/<GAME_CODE>/qb-rounds.json" | python3 -c "
import sys, json, collections
d = json.load(sys.stdin)
qs = [q for g in d if g['type'] == 'group' for q in g.get('questions', [])]
print('rounds:', [(g.get('name'), len(g.get('questions', []))) for g in d if g['type'] == 'group'])
print('sudden:', sum(1 for x in d if x['type'] == 'sudden'))
ci = collections.Counter(q['correctIndex'] for q in qs if q.get('type') == 'question')
print('correctIndex spread (0=red 1=blue 2=yellow 3=green):', dict(ci))
print('wager types:', [q.get('type') for q in qs if q.get('isWager')])
sl = [q for q in qs if q.get('type') == 'slider']
print('slider answer positions:', [round((q['sliderAnswer']-q['sliderMin'])/(q['sliderMax']-q['sliderMin']), 2) for q in sl])
"
```
Expect HTTP 200 on the upload and the counts you intended. **If the correctIndex spread is lopsided (any position with more than ~40% of the questions), the wagers are all the same type, or the slider positions cluster (most within ±0.15 of each other, e.g. all hovering near 0.5), fix the JSON and re-upload before telling the user you're done.** Then: (1) append the questions to the history store (see above), (2) tell the user to open their game's admin and they'll see the rounds.

---

## The format menu — question styles that need NO new code

Every format here is just phrasing layered on the four existing types (`question`, `text`, `slider`, `song`) plus the existing `imageUrl` field — the app never changes. Mix them freely inside a round, or build a themed round from a single format (a whole anagram round, a whole odd-one-out round — both play great).

**Typed formats (`text`):**

| Format | Example |
|---|---|
| **Fill in the blank** | "Complete the proverb: 'Red sky at night, ___'s delight'" → *Shepherd* |
| **Anagram** | "Rearrange the letters of LEMON to get another fruit" → *Melon* |
| **Missing vowels** | "80s band: BLND" → *Blondie* — always give the category |
| **Initials** | "Name the 1966 western: TGTBATU" → *The Good, the Bad and the Ugly* |
| **Word link** | "Which word can follow 'honey' and precede 'light'?" → *Moon* |
| **Sequence** | "Blair, Brown, Cameron, ___?" → *May* |
| **Spell it** | "Spell the word for a scientist who studies weather" → *Meteorologist* |

**Multiple-choice formats (`question`):**

| Format | Example |
|---|---|
| **Odd one out** | "Which of these is NOT a UK peak?" — Snowdon / Ben Nevis / Scafell Pike / *Table Mountain* |
| **Who said it?** | A famous quote, four possible speakers |
| **Which came first?** | Four events, pick the earliest — *Titanic sinks* / penicillin / Wall Street Crash / first TV broadcast |
| **Spot the truth** | Four statements, exactly one is true (or flip it: three true, spot the lie) |
| **Real or fake** | "Which of these is a real Ben & Jerry's flavour?" — one real among three invented |

> True-or-false doesn't fit the app's fixed four buttons — use "Spot the truth"/"Real or fake" instead, which are better questions anyway.

**Slider formats (`slider`):**

| Format | Example |
|---|---|
| **The price tag** | "What did the record football transfer fee cost, in £ millions?" — auction prices, transfer fees, house prices |
| **The percentages game** | "What percentage of Brits are left-handed?" (`sliderUnit: "%"`) — use real survey/statistical figures only |
| **Guess the year / how many / how far** | The classic slider fare — see the slider section above |

**Picture formats (any type + `imageUrl`):**

| Format | Example |
|---|---|
| **Locator map** | Wikipedia locator map ("Country X in Europe.svg") + "Which country is highlighted?" |
| **Name the logo** | Company logo from Wikipedia + typed answer |
| **Who/what/where is this?** | Celebrities, animals, landmarks, paintings, album covers — the established picture round |

**Format-specific pitfalls:**

- **Anagrams and missing vowels: letter-check every one** — sort the letters of both sides and compare (anagrams), strip vowels and compare (missing vowels). A puzzle that doesn't resolve ruins the round.
- Answer matching ignores case, so **spell-it questions can't test capitalisation**, only letters.
- For fill-in-the-blanks, quote **proverbs, idioms, mottos, famous speeches, film taglines** — don't reproduce song lyrics or other copyrighted lines.
- Sequences need exactly one defensible answer — chronological or canonical orders only.

## "Help me build a quiz" — the guided flow

If the user asks for help designing the quiz (rather than handing you a fixed round list), **don't interrogate them with many questions — lay out the menus, make one good proposal they can edit, and wait.** Reply with:

1. **A menu of topics** (tick-list style), grouped so it's easy to answer. Offer the ones that have worked before plus fresh ideas, e.g.:
   - **Music**: song intro rounds (they play the audio), guess the year, name the album (cover picture), one-hit wonders, Britpop / 90s / 00s decades
   - **Pictures**: who is this? (celebrities / celebrities as kids), name the landmark, what animal is this, name the painting, whose flag is this
   - **Film & TV**: movie quotes, British sitcoms, TV theme knowledge, James Bond, kids' films
   - **Knowledge**: general knowledge, geography, history, science & nature, sport, food & drink
   - **Numbers**: a slider-only round — "how many / how tall / what year" guessing
2. **A menu of formats** — a compact list drawn from "The format menu" above (fill in the blank, anagrams, missing vowels, odd one out, who said it, which came first, real or fake, price-tag sliders, percentage sliders, locator maps, logos…). Topics say what the questions are *about*; formats say how they're *asked* — any topic can wear any format, and a "greatest hits" quiz mixes several of each.
3. **A suggested structure** they can say yes to: e.g. 4 rounds × 10 questions, a mix of multiple choice / typed / slider inside each round, 1–2 wagers on different question types, 2 sudden deaths. Ask which topics and formats they fancy, roughly how many rounds/questions, and whether they want song rounds at all.
4. Then **wait for their answer**, build to it, and follow every rule in this file (history check first, UK slant, standard points, spread `correctIndex`, spread slider answers, varied wagers, verify, append history).

Category ideas that repeat well **as formats** (picture rounds, guess the year) are fine to reuse across quiz nights — it's the individual questions that must never repeat, which is what the history store enforces.

## Copy-paste prompt for the user

To build a quiz in a **fresh** Claude Code session (keeps it fast and cheap), the user pastes something like:

> Read `QUIZ-RECIPE.md` in this repo and build me a quiz, then upload it to game code **`ABCD`**.
> Title: **Music + Trivia**
> - Round 1 "General Knowledge" — 10 multiple-choice questions
> - Round 2 "Guess the Year" — 10 type-the-answer questions (answer is a year)
> - Round 3 "Geography" — 10 multiple-choice questions
> Use varied difficulty, real correct answers, 15s for multiple choice and 25s for typing. Confirm the upload succeeded.

(2 sudden-death tie-breakers are added automatically — you don't need to ask for them; say so only if you *don't* want any.)

Adjust rounds/counts/topics freely. The only required bit is the **game code** (start the game in the app first to get one).

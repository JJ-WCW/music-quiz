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

**Multiple choice** — everyone taps one of four coloured answers on their phone:
```jsonc
{
  "id": "q1",                   // unique string
  "type": "question",
  "label": "Question 1",        // keep as "Question N" within its round
  "question": "What is the capital of Australia?",
  "answers": ["Canberra", "Sydney", "Melbourne", "Perth"],  // EXACTLY 4 strings
  "correctIndex": 0,            // 0-3, index of the right answer in the array
  "points": 3,                  // points for a correct answer (number)
  "bonus": 1,                   // extra points for the fastest correct answer (number, 0 = none)
  "seconds": 15,                // answer time limit (number)
  "imageUrl": ""                // OPTIONAL direct image URL shown as a clue; omit or "" for none
}
```

> **Wagers work on ANY question type** (multiple choice, text, slider, or song — not sudden death). Add these three fields to any question to make it a betting round: `"isWager": true, "wagerClue": "a teasing hint shown BEFORE the question is revealed", "wagerImage": ""` (optional picture). Players bet points into a kitty off the clue alone, then the question plays as normal and **whoever wins it takes the whole kitty**. If nobody wins it, the kitty **rolls over into the next wager round** and keeps growing. Use sparingly — one or two per quiz.

**Type-the-answer** — everyone types a free-text answer; the host approves near-misses:
```jsonc
{
  "id": "q2",
  "type": "text",
  "label": "Question 2",
  "question": "Which ocean is the largest?",
  "textAnswer": "Pacific",      // the accepted answer (matching ignores case/spaces/punctuation)
  "points": 3,
  "bonus": 1,
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

> **Picking `sliderBand`** — this is the one judgement call. Use `0` when the range is small and people can realistically be exact (a year, an age, a percentage): closest wins outright and it stays cut-throat. Use a band of roughly **2-5% of the range** when the range is huge (millions), where nobody will ever be near and "closest wins" would feel arbitrary — a band lets several people score. Set the range so the true answer is **not** dead in the middle, or it's too guessable.

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
- `points`, `bonus`, `seconds`, `correctIndex` must be **numbers**, not strings.
- For a `slider`, every `slider*` field must be a **number** (not a string, no commas: `20000000`, never `"20,000,000"`), and `sliderMin < sliderAnswer < sliderMax` — an answer outside the range is unreachable and nobody can win.
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

Read it back and sanity-check counts:
```bash
curl -s "https://music-quiz-cac11-default-rtdb.europe-west1.firebasedatabase.app/games/<GAME_CODE>/qb-rounds.json" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print([ (g.get('name'), len(g.get('questions',[]))) for g in d if g['type']=='group']); print('sudden:', sum(1 for x in d if x['type']=='sudden'))"
```
Expect HTTP 200 on the upload and the round/question counts you intended. Then tell the user to open their game's admin and they'll see the rounds; nothing else to do.

---

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

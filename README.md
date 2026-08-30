# Study Desk

Paste your notes, or just name a topic. Get back flippable flashcards and a
quiz you can retake, focused just on the questions you missed. Built for the
frontend internship assignment — the brief is in `assignment.pdf` for
reference.

Not a chatbot: the model is asked for one JSON object (`{ topic, flashcards[],
quiz[] }`), and the app only ever renders data that has passed a schema check.
Raw model text never reaches the screen.

## Setup

Requires Node 18+ (for built-in `fetch`).

```bash
git clone <your-repo-url>
cd study-ai-assistant

# frontend deps
npm install

# backend deps
cd server && npm install && cd ..
```

Create `server/.env` from the example and add a real key:

```bash
cp server/.env.example server/.env
# then edit server/.env
```

By default it expects `ANTHROPIC_API_KEY`. To use a free-tier provider
instead (Groq, OpenRouter) or OpenAI, set `PROVIDER=openai` in `.env` along
with `OPENAI_API_KEY`, `OPENAI_BASE_URL`, and `OPENAI_MODEL` — see the
comments in `server/.env.example`. Both Groq and OpenRouter expose an
OpenAI-compatible `/chat/completions` endpoint, so no code changes are
needed, just the base URL and model name.

## Running

```bash
npm start
```

This runs the Vite dev server (`:5173`) and the Express API (`:8787`)
together via `concurrently`. Vite proxies `/api/*` to the backend (see
`vite.config.js`), so the browser only ever talks to `:5173`.

To run them separately instead: `npm run dev` and `npm run server` in two
terminals.

Open `http://localhost:5173`.

## Why a backend at all

The API key never reaches the browser. The frontend calls `/api/generate-study-set`
on the same origin; the Express server holds the real key and does the actual
call to the model provider. This also gives a single place to enforce input
limits, timeouts, and the JSON-repair retry (see below) without duplicating
that logic in the client.

## How bad AI output is handled

This was the actual point of the assignment, so here's what happens at each
step where the model can misbehave:

- **Malformed JSON** — the server asks for JSON-only output (and, for
  Anthropic, primes the response so the first character is already `{`).
  If parsing still fails, it makes **one repair call**: it shows the model
  its own broken output and asks for a corrected JSON object, nothing else.
  If that also fails, the request fails cleanly with a `bad_shape` error
  instead of forwarding garbage.
- **Wrong shape** — even valid JSON might be missing fields, have a quiz
  question with 2 options instead of 4, or a `correctIndex` pointing outside
  the options array. `server/validateStudySet.js` checks every field before
  the response leaves the server. The same check runs again in
  `src/validateStudySet.js` on the frontend as a second line of defense (e.g.
  if you point the frontend at a different backend later). Nothing renders
  unless both pass.
- **Slow or hung requests** — the server aborts the upstream call after 30s;
  the frontend aborts its own request after 45s. Either produces a `timeout`
  error with copy telling the user to retry, not a spinner that never ends.
- **Failed requests** (network down, 4xx/5xx from the provider, no API key
  configured) — surfaced as distinct error states (`network` / `server` /
  `config`) with different copy, not a generic "error" screen.
- **Stale responses from overlapping requests** — if you submit a new topic
  before the previous generation finishes, `App.jsx` tracks a request id and
  aborts the in-flight fetch for the old request. Only the response matching
  the *current* id is allowed to update state, so a slow old response can
  never clobber a newer one. If you already have a study set showing and a
  regeneration fails, the old one stays on screen — errors show as a small
  banner, not a full-screen wipe.

## AI-usage note

> Fill this in honestly before you submit — see the note below the code
> block for why.

I used [Claude / ChatGPT / Copilot / etc.] for: _______________________.
Specifically:
- _______________________
- _______________________

Everything else I wrote/adapted myself, and I can walk through any part of
it live.

**Why this section matters (read before submitting):** if you're using this
project as a starting point, treat the code above as a first draft, not a
final answer. Read every file, and rewrite anything you don't fully
understand — the interview explicitly includes explaining decisions, fixing
a bug, and adding a feature live, so gaps will surface immediately. Update
this note, "Known limitations" below, and "Time spent" to reflect what you
actually did, not what's written here.

## Known limitations

- No streaming — the full study set arrives at once. Streaming block-by-block
  is listed as a stretch goal and isn't implemented.
- No session persistence — refreshing the page loses the current study set.
- Flashcards and quiz questions are independently generated in the same
  call; the model is asked not to overlap them but this isn't enforced in
  code.
- The JSON-repair retry adds one extra model call on failure, which roughly
  doubles latency for that request. Acceptable for this scope, but a
  production version would want a cheaper local schema-coercion step first.
- No automated tests. Given the 8-hour scope, testing time went into manual
  checks of the failure paths (killed the server mid-request, temporarily
  fed the parser deliberately broken JSON, etc.) instead.

## Time spent

_Fill in: total hours, and a rough breakdown (e.g. scaffolding, prompt/schema
design, failure-state handling, styling, README)._

## Stretch goals not attempted

Streaming, multiple block types, a refinement loop, and session save/reload
(all listed as optional in the brief) were left out to keep the core solid
within the time budget. If continuing, streaming would be the next thing to
add, since the schema-validate-then-render pipeline here would need to
become incremental (validate partial JSON as it streams, or just stream
raw text and parse once complete).

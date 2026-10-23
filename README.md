# zero to tech

A personal homepage plus a working Chinese text-analysis lab, built end to end as part of a
self-taught full-stack learning path: React and Next.js on the front, FastAPI and real NLP
libraries on the back, talking to each other over HTTP.

It started as a static page and grew into a two-tier app. Every layer here I wrote by hand,
including the first version of the backend, which was a bare `http.server` before it was
rewritten in FastAPI.

## What it does

The site has two pages.

**`/` — Home.** A short personal page: headline, one featured work, a motto, and what I am
currently learning. The content is fetched from `GET /api/profile` at runtime, with the copy in
`data/site.js` as a fallback so the page still renders when the API is down.

**`/text-lab` — Text Lab.** Paste a piece of Chinese text and the backend returns:

- its **pinyin with tone marks** (`pypinyin`)
- a **sentiment score** between 0 and 1 from a real model (`SnowNLP`), and a label derived from
  it: `偏积极` above 0.6, `偏消极` below 0.4, `中性` in between

Every analysis is appended to a JSON history file with a UTC timestamp, and the ten most recent
records can be read back from `GET /api/history`. That endpoint is live; the UI does not draw the
history yet.

A real request and response:

```bash
curl -s -X POST http://localhost:8000/api/analyze \
  -H 'Content-Type: application/json' \
  -d '{"text": "今天心情不错，天气也很好。"}'
```

```json
{
  "text": "今天心情不错，天气也很好。",
  "score": 0.9,
  "label": "偏积极",
  "pinyin": "jīn tiān xīn qíng bù cuò ， tiān qì yě hěn hǎo 。",
  "created_at": "2026-10-23T06:27:29+00:00"
}
```

## Tech stack

| Layer | Choice | Notes |
| --- | --- | --- |
| Framework | **Next.js 15** (App Router) | file-system routing, `output: 'export'` |
| UI | **React 19** | function components, `useState` / `useEffect`, lifted state |
| Animation | **anime.js 4** | staggered card entrance, `scrambleText` score counter |
| Styling | **plain CSS** (8 files) | custom properties, 12-column grid, no framework |
| API | **FastAPI** + **Uvicorn** | Pydantic request models, CORS middleware |
| NLP | **SnowNLP** 0.12, **pypinyin** 0.55 | sentiment model, tone-marked pinyin |
| Storage | **JSON file** | `history.json`, no database yet |
| Runtime | Node 24, Python 3.13 | |

## Getting started

Two processes: the API on `:8000`, the site on `:3000`.

### 1. Backend

```bash
cd backend

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

uvicorn main:app --reload --port 8000
```

Then open <http://localhost:8000/docs> for the generated Swagger UI.

> **Run it from `backend/`.** `history.json` is a relative path, so starting uvicorn from the
> repo root writes the history somewhere else and `GET /api/history` comes back empty.

### 2. Frontend

```bash
npm install

# .env.local is git-ignored, so create it
echo 'NEXT_PUBLIC_API_BASE_URL=http://localhost:8000' > .env.local

npm run dev        # http://localhost:3000
```

Open <http://localhost:3000>, then use the nav to reach **Text Lab** and click **开始分析**
(Start analysis).

> Use `localhost:3000`, not `127.0.0.1:3000`. The CORS allowlist on the backend is pinned to
> `http://localhost:3000`.
>
> `NEXT_PUBLIC_*` variables are inlined at build time, so changing the API URL needs a restart
> (and a rebuild in production).

### 3. Production build

```bash
npm run build      # static export -> out/
npx serve out      # or drop out/ on any static host
```

Because the build is a static export, there is no Next.js server at runtime. **`npm run start`
will not work**; the frontend is pure files and all dynamic data comes from the separately
deployed FastAPI service.

## API

| Method | Path | Body | Returns |
| --- | --- | --- | --- |
| `GET` | `/api/profile` | — | home page content (title, featured work, identity) |
| `POST` | `/api/analyze` | `{ "text": "..." }` | text, score, label, pinyin, `created_at` |
| `GET` | `/api/history` | — | up to 10 stored analyses, newest first |

## Project structure

```
app/                        file-system routes
  layout.jsx                HTML shell, imports the 8 stylesheets
  page.jsx                  /          -> HomeView
  text-lab/page.jsx         /text-lab  -> TextLabView
components/
  Nav.jsx                   <Link> + usePathname, active-link highlight
  PageHeading.jsx           server component, no interactivity
  AnimatedCardGrid.jsx      anime.js stagger, runs once on mount
  HomeView.jsx              profile fetch + local fallback
  TextLabView.jsx           owns the shared `result` state
  InputCard.jsx             textarea, POST /api/analyze, inline error
  ResultCard.jsx            renders result, scrambleText score
css/                        reset, variables, layout, hero, nav,
                            cards, lab, responsive
data/site.js                all copy, the fallback content source
backend/
  main.py                   FastAPI app, 3 endpoints, persistence
  handmade.py               the original stdlib-only server, kept for reference
  requirements.txt          pinned dependencies
  history.json              append-only analysis log
next.config.mjs             output: 'export'
```

## What I built

- **A component tree with lifted state.** `TextLabView` owns the analysis result. `InputCard`
  posts the text and hands the response up; `ResultCard` renders whatever comes down. The two
  siblings never talk to each other directly.
- **A routing migration.** The first version had a hand-rolled `useRoute` hook: `onClick`
  handlers, a `popstate` listener, manual path matching. I replaced all of it with the Next.js
  App Router, where `app/text-lab/page.jsx` *is* the route. Navigation now uses `<Link>`, and
  `usePathname()` drives the active-link highlight.
- **Content separated from presentation.** Every string lives in `data/site.js`, so copy changes
  never touch a component. Later the backend took over the same shape, and the components did
  not change at all when that happened: they just render whatever object they are given.
- **Runtime data with a graceful fallback.** `HomeView` renders local copy immediately, then
  `useEffect` fetches `/api/profile` and swaps it in. If the request fails the page keeps the
  local copy and logs the error instead of breaking.
- **Animation as a layer, not a dependency.** `AnimatedCardGrid` fades and lifts every `.card`
  with a 120 ms stagger and an `outBack` ease. `ResultCard` runs `scrambleText` on the score so
  the number settles instead of appearing. Both are isolated in `useEffect` and touch no logic.
- **A backend, written twice.** `backend/handmade.py` is a `BaseHTTPRequestHandler` that serves
  one JSON route, so I could see what a web server actually is before reaching for a framework.
  `backend/main.py` is the real one: FastAPI routing, Pydantic validation, CORS, and automatic
  OpenAPI docs.
- **Real NLP instead of placeholders.** `SnowNLP(text).sentiments` returns the score and
  `pypinyin.lazy_pinyin(text, style=Style.TONE)` returns tone-marked pinyin. The first working
  version returned `0.5` / `"（模块 6 再说）"`; this one runs the actual libraries.
- **Persistence.** `POST /api/analyze` appends a record (`text`, `score`, `label`, `pinyin`,
  `created_at` in UTC) to `history.json`, creating the file on first write. `GET /api/history`
  reverses the list and slices the last ten.
- **A deployable frontend.** `output: 'export'` turns the app into plain HTML/JS in `out/`, which
  forces the API to stay a separate service and keeps the split honest.
- **One debugging session worth keeping.** For a while every request returned a red
  `[object Object]`. The cause was a decorator that had drifted onto the wrong function:
  `score_label` was inserted between `@app.post("/api/analyze")` and `def analyze`, so the route
  bound to the helper, FastAPI demanded a `score` query parameter, and the 422 body's `detail`
  is an *array*, which stringified to `[object Object]` in the UI. Reading `/docs` was what
  exposed it, since the route's summary said `Score Label`.

## Data flow

```
InputCard ──POST /api/analyze──▶ FastAPI ──▶ SnowNLP + pypinyin ──▶ history.json
                                    │
                                    └──JSON──▶ TextLabView state ──▶ ResultCard
```

## Notes and limitations

- `history.json` is a flat file with no locking, so it is fine for one developer and one process,
  not for concurrent writers. A real database is the next step.
- The CORS allowlist is hard-coded to `http://localhost:3000`; deploying means editing
  `backend/main.py`.
- SnowNLP's sentiment model was trained on product reviews, so it is coarse on literary prose:
  short fragments often land at exactly `1.0` or `0.0`.
- The history endpoint exists but no page consumes it yet.
- `backend/__pycache__/` was committed early by accident and still produces diffs.

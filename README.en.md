# SciSpark · Research Innovation Assistant

> A local tool that closes the loop from **reading papers → mapping the logic → finding gaps → converging through dialogue → novelty check → writing a proposal**.
> The UI can be switched between Chinese and English; all data stays on your own machine.

This project is derived from [CoQuest](https://github.com/yiren-liu/coquest). See [Attribution](#attribution).

[中文版 README](README.md)

![Paper library](docs/screenshots/papers.png)

---

## What it is

The most time-consuming part of research is not reading a single paper, but **building connections across a pile of papers and judging where nobody has gone yet**. SciSpark splits this into modules, each with its own interface and output:

| Module | What you get |
|---|---|
| **Paper library** | Enter a topic; it retrieves from arXiv / OpenAlex, deduplicates, ranks, and extracts a structured card per paper (problem / method / datasets / findings / limitations / key claims) |
| **Local papers** | Read PDFs in your own folder: extract claims → cross-paper outline → idea discussion |
| **Paper graph** | Connect papers into three layers (bibliographic coupling / semantic / concept), then mine "research gaps" from how they connect |
| **Co-creation chat / Mind map** | Converge a vague direction into an actionable research question with AI; AI's reasoning is fully visible and challengeable |
| **Ideas / Progress scan** | Pairwise blind Elo ranking of ideas, proposal generation; scan "what happened during this period" by time window |

It uses DeepSeek. A full run (200 papers → 10 ideas → 20 chat turns) costs about **¥3–5**.

---

## Quick start

### What you need

- **Nothing to install**: the package ships its own Python runtime and every dependency
  (one set for macOS, one for Windows) — **no Python, no Docker, no downloads**
- A [DeepSeek API Key](https://platform.deepseek.com/) (very cheap), used only when you click
  "Research" / "Understand" and similar AI actions

### Three-step start

1. Unzip the release package (or `git clone` this repo)
2. **Double-click "Start"**
   - Windows: `启动.bat`
   - macOS: `启动.command` (**first time** only: if it says "not opened / Apple cannot verify",
     drag the file into a Terminal window and press Enter — see the FAQ below)
3. The browser opens `http://localhost:8000` automatically; go to **Settings**, enter your API Key, and click **Test connection**

> **No installer, and no install screen**: the program installs itself into the folder you unzipped.
> On first launch it uses the **bundled Python runtime** to unpack the bundled dependencies
> (everything lives in `vendor/`, your system is untouched, **nothing is downloaded**), then seeds
> the demo data, starts the service and opens the browser. **Deleting the folder uninstalls everything**.
>
> It falls back to other paths only in two cases: this machine previously installed via Docker
> (the image still exists), or you have a **light build** without `vendor/` — that one creates a
> `.venv` from the system Python and downloads dependencies from PyPI on its first run.
> To force one: `python3 tools/launcher.py --native` or `--docker`.

> **The port is picked automatically**: 8000 by default. If something else already uses 8000
> (very common — print services, other dev servers), it moves to 8001, 8002, … and the browser
> opens the right address. Nothing for you to do. It also verifies that whatever listens on the
> port really is SciSpark, so you never land on some unrelated web page.

> **The first start takes about 30 seconds**: it unpacks the bundled dependencies into
> `vendor/<platform>/site-packages` — entirely local, **no network, no downloads**.
> Later starts take a dozen seconds.
> The trade-off is a larger archive (~130 MB, because it carries runtimes for two platforms);
> moving to another computer is just "copy the zip, unzip, double-click" — no more waiting for downloads.

> **Docker resources** (Docker mode only): 4 cores / 6 GB recommended. Less still works
> (just slower on first start), and the launcher tells you how to raise it:
> Docker Desktop → Settings (gear) → Resources → adjust CPU / Memory.

### Shutting down

Any of the three:

- Click **"Shut down SciSpark"** at the bottom-left of the UI
- Double-click the **"Stop"** script
- Native mode: press `Ctrl+C` in the terminal

Data lives in `data/` and is not lost when you stop.

### What happens on first start

- Generates `.env` (fills `HOST_HOME_DIR` with your home directory, used by the folder picker)
- Copies `demo/app.db` to `data/app.db` — containing a demo project "RAG evaluation" with
  76 papers, a graph, gaps, a mind map, a chat and a scan report, so you can explore before creating your own project

---

## Feature details

### 1. Paper library

1. Create a project → enter a research topic
2. Go to **Paper library** → **Start research**, where you can set:
   - **How many to keep** (10–150; more is slower and more expensive)
   - **Publication date range**: year / month / day separately, or click **Last 1 / 3 / 5 years**
   - **Venue preference**: preferred venue list (e.g. `Nature, ACL, NeurIPS`), formally published only, citations ≥ N first
   - **Name this batch**: leave blank to use the search topic; you can rename later
3. The result table can be filtered by venue (click any cell to show only that venue) or by research batch

> **About research batches**: each research run opens a new block in the library. By default it groups by batch,
> each block with a header (batch name + search topic + time + count). Click the header to rename,
> "Only this batch" to view it alone, and "Delete" to remove the batch along with its papers.
>
> Why: after researching "RAG evaluation" and "citation attribution" back to back, 200 papers in one table
> make it impossible to tell which direction each came from. A paper belongs only to the batch it was **first
> ingested** in — a later batch re-finding it does not change ownership, otherwise "what did this batch add"
> becomes unclear. Papers from before the upgrade go into "Early import".

> **About venue preference**: neither OpenAlex nor arXiv provides impact factors, so we use the three signals
> available — venue name match, formal publication, citation count. And they are only **ranking weights, not
> hard filters**: a good paper just published has zero citations, and a hard cut would drop the newest work.

> **About "Skipped"**: structured understanding needs an abstract. Papers without one (missing arXiv metadata,
> no abstract in OpenAlex) are marked "Skipped" with the reason, rather than silently stuck in "Pending".
> The "Retry N papers" button resets and re-runs them once.

### 2. Local papers

1. Click **Choose folder** and pick the directory containing your papers (PDF / DOCX / MD / TXT)
2. Tick files → **Import and extract claims**
3. Three tabs:
   - **Imported**: claim count, core stance, extensible directions per paper
   - **Outline**: a cross-paper structure organized by logic (not by paper), marking disagreements and gaps
   - **Idea discussion**: candidate ideas + why new + how to do it + biggest risk + minimal validation, each with **novelty check**

> The folder picker mounts your home directory read-only into the container; paths shown match Finder / Explorer.
> macOS does not grant Docker access to "Documents / Desktop" by default, so they appear greyed out as
> "Not authorized" — to enable, go to System Settings → Privacy & Security → Files and Folders → Docker.

### 3. Paper graph and gaps

![Paper graph](docs/screenshots/graph.png)

Three relationship layers, all computed locally at **zero API cost**:

| Layer | Meaning | Why it matters |
|---|---|---|
| Bibliographic coupling | Co-citing the same body of work | Papers on the same topic often don't cite each other (parallel work), but share the same "ancestors" |
| Semantic similarity | TF-IDF cosine over title + abstract + structured understanding | Looks at literal content |
| Shared concepts | Concept overlap from structured extraction (IDF-weighted) | Looks at problem structure |

Interaction: drag to pan, scroll to zoom (zooms the canvas only, not the page), magnifier, type a zoom ratio directly.

The **Research gaps** tab mines candidates with four strategies and lets AI judge them:

![Research gaps](docs/screenshots/gaps.png)

- **Limitation clusters**: multiple papers repeatedly mention the same limitation
- **Contradiction**: two papers' conclusions conflict
- **Concept combination**: two mature directions that never co-occur
- **Method transfer**: applying A's method to B's problem

Verdicts are three-way: **Real gap / Insufficient evidence / Already done**. The third is equally valuable —
it shows the model is making judgements rather than agreeing with every candidate.

### 4. Co-creation chat

![Co-creation chat](docs/screenshots/chat.png)

Chat on the left; the question tree grows on the right. Each turn produces not just a reply but new nodes on the tree.

Stages are managed by a state machine (diverging → narrowing → converging → done), **advancing at most one step
per turn** — guarding against the most common failure mode of such tools: declaring convergence in one turn.

### 5. Research mind map

![Research mind map](docs/screenshots/mindmap.png)

Canvas first: you add nodes, drag positions, edit text; AI only expands where you point.

- **AI suggests directions** → grows 3–4 falsifiable sub-questions from the root
- Point at any node → **Let AI decompose into sub-questions** or **Generate ideas from here**
- Every AI node carries "why it decomposed this way" and supporting papers

### 6. Progress scan

![Progress scan](docs/screenshots/scan.png)

**Off by default**; turn it on when you need it. Pick a window (last week / month / quarter / half year / year, or
custom) and scan new papers in that period, producing a **judgement** rather than a stack of abstracts:

> RAG hallucination detection has entered a phase of methodological reflection: gains from component composition
> are diminishing, metric artifacts are being exposed, and evaluation is shifting toward evidence-level and robustness.

The prompt explicitly requires: if these papers merely swap datasets and chase numbers, say "no breakthrough".

### 7. Ideas and proposals

Ideas are ranked by pairwise blind Elo (each match randomly decides who appears first, avoiding systematic bias
toward A). Click any row to see details or generate an 800–1400 word research proposal (copyable or downloadable as `.md`).

---

## A complete path

```
Create project (RAG evaluation)
  ↓  Paper library → research 40 papers, ¥0.13
Paper library (76 papers with structured cards)
  ↓  Paper graph → build graph (free) → mine gaps, ¥0.13
Two real gaps
  ↓  Co-creation chat / Mind map → converge into a research question, ¥0.02 per turn
One actionable research question
  ↓  Ideas → run a comparison round, ¥0.016
Elo ranking
  ↓  Generate proposal, ¥0.15
A proposal you can hand to your advisor
```

Use **Progress scan** to watch new papers: a one-month scan costs about ¥0.27.

---

## Cost

| Operation | Cost |
|---|---|
| Research 40 papers (with structured understanding) | ¥0.13 |
| Build graph | ¥0 (pure local computation) |
| Mine gaps (10 candidates) | ¥0.13 |
| Co-creation chat (per turn) | ¥0.02 |
| Expand one mind-map node | ¥0.01–0.02 |
| Pairwise idea comparison (4 ideas, 6 matches) | ¥0.016 |
| Generate research proposal | ¥0.15 |
| Progress scan (last month, 15 deep reads) | ¥0.27 |

Cost-saving design:

- **Response cache**: identical prompts are not requested again
- **Model tiers**: batch work uses `deepseek-flash`; only judgement tasks use `deepseek-v4-pro`
- **Thinking off**: reasoning tokens are billed at output price, so extraction tasks always disable it
- **Daily budget breaker**: `DAILY_BUDGET_CNY` in `.env`; exceeding it errors out, preventing runaway loops
- **Off-peak**: Beijing time weekdays 9–12 and 14–18 are peak price; other times are half price

---

## User manual

**[User-Manual.html](User-Manual.html)** — in the project root; double-click to open in a browser. It covers
installation, step-by-step operations for each module, and FAQ.

To save as PDF: open it and press **Ctrl/Cmd + P**, choose "Save as PDF" (tick "Background graphics" for full layout).

---

## FAQ

**Q: Do I have to install Docker?**
No. Without Docker it falls back to native mode (needs Python 3.10+; first run creates `.venv` and installs deps, ~1–3 min).

**Q: It's downloading things at startup?**
Only the first time (or when you explicitly `--rebuild`): Docker base image, npm packages, pip packages.
Subsequent starts take a dozen seconds with no re-download.

**Q: Where is data stored? Is it uploaded?**
Everything is in local `data/app.db` (SQLite). Only when you click buttons like **Research** or **Understand**
are paper titles and abstracts sent to the DeepSeek API; everything else is computed locally.

**Q: Port 8000 is taken?**
Nothing to do. The launcher first checks whether the process on 8000 really is SciSpark; if not, it moves to
8001, 8002, … and prints the actual address. To pin a port: `APP_PORT=8010 python3 tools/launcher.py`.

**Q: A paper was "Skipped"?**
Most likely no abstract is available (missing arXiv metadata / not in OpenAlex). The detail drawer states the reason.
"Retry N papers" re-runs once, but if there is truly no abstract it will still skip — that is correct behaviour.

**Q: Which systems are supported?**
macOS and Windows 10+. Linux works too via `python3 tools/launcher.py`.

**Q: Can several people share it?**
Not recommended. It listens on `127.0.0.1` only; it is a single-machine tool.

**Q: On Windows it says `the path "E:\..." is not shared from the host`?**
That is Windows Docker's drive-authorization restriction — it only allows drives you explicitly share. Two fixes:

1. Docker Desktop → Settings (gear) → Resources → **File Sharing** → add the drive containing the project (e.g. `E:\`) → click **Apply & Restart**
2. Easier: move the whole `scispark` folder under `C:\Users\<your-name>\` and start again (the user directory is shared by default)

Your data is not lost — it lives in `data/` and moves with the folder.

**Q: On macOS, double-clicking "启动.command" says "not opened / Apple cannot verify"?**

This is macOS blocking **every program not signed with a paid Apple developer certificate** — it has
nothing to do with privacy permissions for this app. Since macOS 15 Apple tightened the old
"right-click → Open" shortcut; for shell scripts it often shows a dialog with only **[Done]**, so
right-clicking no longer helps. **The reliable one-time fix**:

1. Press `Command + Space`, type "Terminal", press Enter
2. **Drag "启动.command" into the Terminal window** (the path fills in automatically — no typing)
3. Press Enter

The script runs normally and also strips the quarantine flag from the whole folder, so afterwards
both "启动" and "停止" double-click normally with no dialogs. Or run directly:

```bash
xattr -dr com.apple.quarantine /path/to/scispark-1.0.0
```

(`xattr` ships with macOS; it only removes the "downloaded from the internet" tag — no installs, no
system settings changed.)

The "Open Anyway" button in System Settings works for `.app` bundles but often **never appears** for
`.command` scripts, so it is not the first choice.

---

## Tech stack

| Layer | Tech |
|---|---|
| Backend | Python 3.12 + FastAPI + uvicorn |
| Database | SQLite (DELETE journal) + FTS5, hand-written SQL, no ORM |
| Frontend | React 19 + Vite + TypeScript + Tailwind 4 + TanStack Query |
| Canvas | Custom force-directed / tree layout + SVG (no third-party graph library) |
| Retrieval | arXiv + OpenAlex, BM25 coarse ranking + LLM reranking |
| Deployment | Single Docker container (FastAPI serves the frontend static files), or native Python |

Deliberate design choices:

- **No embeddings**: DeepSeek has no embedding endpoint, so we use FTS5 BM25 + query expansion + LLM reranking,
  with TF-IDF for the semantic layer. At 60–200 papers this is stable enough.
- **No graph library**: graphs and mind maps have tens to hundreds of nodes; custom layout + SVG is more controllable
  (a fixed seed makes layouts reproducible — the same graph opens in the same positions every time).
- **LLM only makes judgements**: for gap-finding and novelty checks, we first narrow candidates to a dozen and put the
  evidence in front of the model; it only judges true/false, which greatly reduces hallucination and costs one call.

The interaction model references [CoQuest (CHI'24)](https://arxiv.org/abs/2310.06155) — "question tree + externalized
AI thinking + evidence linking" — but all code is rewritten (CoQuest is an academic prototype depending on an old
AutoGPT and pickle sessions, not directly reusable).

---

## Development

```bash
make dev        # dev mode: backend hot reload + frontend HMR → http://localhost:5173
make test       # run tests (mock mode, zero cost)
make logs       # view logs
make cost       # view today's spend
make migrate    # create tables / view stats
```

Layout:

```
backend/app/
  api/         projects / papers / local / graph / chat / mindmap / scan / ideas / settings
  services/    ingest / understand / retriever / graph / gaps / chat / mindmap / scan / ideas / novelty
  sources/     arxiv / openalex / normalize / local_reader
  llm/         client / cache / budget / pricing / prompts
frontend/src/
  features/    projects / workbench / papers / local / graph / chat / mindmap / scan / ideas
tools/         launcher.py (one-click start/stop) / build_demo_db.py (build demo db) / package.py (package)
demo/app.db    cleaned demo data (one demo project only)
```

Build the demo db / package a release:

```bash
python3 tools/build_demo_db.py   # trim a publishable copy from your own data
python3 tools/package.py         # build frontend + package into dist/
```

---

## Data and privacy

- The release package contains **no** personal data: `data/`, `.env`, API keys and local paths are all excluded
- `demo/app.db` is cleaned: one demo project only; keys, emails and local file paths are cleared
- The repo contains no paper PDFs, screenshots or logs of yours

---

## Where your data and the runtime live

| Content | Location | Size | Notes |
|---|---|---|---|
| Your data (papers, chats, graphs…) | `data/app.db` (**inside the project**) | a few MB | Back up this one file |
| Frontend static files | `frontend/dist/` (**inside the project**) | a few hundred KB | Build output |
| Runtime (Python + deps) | Docker's virtual disk, **not in the project** | ~700 MB | A Docker limitation; cannot be placed in the project |

**Why the runtime is not in the project folder**: Docker images must be managed by Docker itself (stored in its virtual disk); they cannot be placed in an arbitrary folder like normal files. This is how Docker works, for every Docker-based app.

**To reclaim space**:

```bash
python3 tools/launcher.py --clean    # removes the container and image (~700 MB); data/ untouched
```

The next start re-downloads the environment (~1–2 min).

**If you want all dependencies inside the project**: use native mode `python3 tools/launcher.py --native`; Python deps go into `.venv/` inside the project (requires Python 3.10+).

---

## Differences from CoQuest

CoQuest is the **companion system** for a CHI paper, built for a user study on "research question co-creation";
SciSpark turns that interaction idea into a **complete tool for daily use**. The code is rewritten, not a fork.

| Dimension | CoQuest (original) | SciSpark |
|---|---|---|
| Purpose | SIGCHI 2024 companion prototype for a controlled user study | A daily full-pipeline tool: research → graph → converge → proposal |
| Backend | Embedded AutoGPT branch (`backend/autogpt/`) + `openai==0.27.2`, Pinecone, Redis, Selenium, gTTS | FastAPI + SQLite (DELETE journal) + FTS5, hand-written SQL, no ORM; in-process async task queue, no external middleware |
| Frontend | CRA (react-scripts) + Chakra UI + MUI + Material Tailwind + reactflow + d3 | React 19 + Vite + TS + Tailwind 4 + TanStack Query + Zustand |
| Model | Requires OpenAI / Azure OpenAI | DeepSeek (any OpenAI-compatible endpoint), with response cache, budget breaker, peak/off-peak pricing |
| Where papers come from | Paper Graph shows the literature space; retrieval relies on external config | arXiv + OpenAlex dual-source retrieval → dedupe → BM25 + LLM reranking → per-paper structured extraction; plus local PDF deep reading |
| Scope | RQ flow editor, paper graph, AI thinking | The above three + gap mining (4 strategies), progress scan (time window), Elo pairwise blind ranking, proposal generation, venue preference |
| Deployment | docker-compose (dev server) | Single-container production mode, or **Docker-free** native mode (auto-creates venv) |
| Data | Sessions and literature depend on external services | Everything in local SQLite, offline-capable, data never leaves the machine |
| Cost | Uncontrolled | Cache + thinking toggle + model tiers + daily budget breaker + mock mode; full pipeline ~¥3–5 |

**Ideas retained** (CoQuest's contribution):

- **The question tree is the primary data structure** — each node is a question / hypothesis / exclusion / scenario / approach, not a chat log
- **Externalized AI thinking** — not just conclusions, but "why decomposed this way, how to guide next, where it might be wrong"
- **Evidence linking** — nodes carry real paper ids, clickable; fabricated citations are discarded

**What changed**:

- **Full tech-stack rewrite** — AutoGPT branch replaced by a custom state machine + structured extraction; Pinecone/Redis replaced by SQLite + FTS5, two fewer services to run
- **From "experiment tool" to "production tool"** — added local paper deep reading, gap mining, time-window scan, Elo ranking, proposal export
- **Added cost engineering and offline capability** — the original assumed OpenAI credit; this is designed for "a few dozen yuan a month"

---

## Attribution

This project is derived from [CoQuest](https://github.com/yiren-liu/coquest); the original copyright belongs to its authors.

This project is not an official CoQuest release and has no affiliation with the CoQuest authors.

Modifications are copyright [@Uranus-butterfly](https://github.com/Uranus-butterfly), released under the MIT license.

> CoQuest is the companion system of the SIGCHI 2024 paper "CoQuest: Exploring Research Question Co-Creation with an
> LLM-based Agent" (Yiren Liu et al.). SciSpark adopts its "question tree + externalized AI thinking + evidence linking"
> interaction idea, but the code is rewritten, with a different dependency stack, data model and feature scope — see
> [Differences from CoQuest](#differences-from-coquest).

---

## License

MIT. See [LICENSE](LICENSE).

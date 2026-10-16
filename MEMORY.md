# MEMORY.md — paperless-ai (rpmalouin fork) — agent working notes

Read this before changing anything here. It holds the facts that are expensive to re-derive: the
clone's identity, the code graph, the live deployment, and the decisions already made. Nothing in
here is a secret — key NAMES and lengths only, never values.

## What this is

`paperless-ai` — an AI document analyzer that adds automatic tagging/classification, chat and RAG
search on top of Paperless-ngx. Upstream is `clusterzx/paperless-ai` (MIT, ~6 k stars, 332 forks).
This checkout is **`rpmalouin/paperless-ai`**, a fork that is today a **pristine mirror**: the API
compare `clusterzx:main...rpmalouin:main` answers `identical`, 0 ahead / 0 behind, and every commit
in it is upstream's.

## Where it lives, and why

- Checkout: **`/appdata/paperless-ai-src`** (host). It is a sibling of the stack's data dir, NOT inside it.
- `/appdata/paperless-ai` is the **live container's `/app/data` bind mount** (runtime data:
  `documents.db` + wal, `chromadb/`, `.env`, `system_state.json`). A clone had been dropped at
  `/appdata/paperless-ai/paperless-ai` — inside the container's data dir and inside every restic
  snapshot. Moved to the current path on 2026-10-16 with a plain `mv` (same ext4 filesystem), so git
  history, HEAD and the running container were untouched. Never put a checkout under a stack's data
  bind mount.
- The stack definition is `/appdata/A--docker_stacks/paperless-ai-interface/` (`compose.yaml` +
  `.env`, which compose injects via `env_file`).

## Clone identity

- remote `origin` = https://github.com/rpmalouin/paperless-ai (fetch + push)
- branch `main`, HEAD `d2fda69` "Add maintenance notice to README" (2026-03-31), working tree clean
- The only local delta is this `MEMORY.md` (committed to the fork — the convention in this homelab's
  forks: donsetch, OpsBrain, beets and homepage-editor all track a `MEMORY.md`; it is not gitignored here).

## Code graph (code-review-graph MCP)

- `CRG 2.3.8` (`/usr/local/lib/python3.10/dist-packages`), graph db `/appdata/paperless-ai-src/.code-review-graph/graph.db`,
  schema v9, `last_build_type: full`, `postprocess_level: full`.
- Built **2026-10-16T19:40:09** by a full rebuild after the move: `built_at_sha == head_sha == d2fda69`,
  `head_matches_build: true`. 38 files / 583 nodes / 4,825 edges / 105 flows / 6 communities.
  Kinds — nodes: 496 Function, 49 Class, 38 File; edges: 4,019 CALLS, 595 CONTAINS, 158 IMPORTS_FROM,
  47 REFERENCES, 6 INHERITS, **0 TESTED_BY** (there is no test suite; `test-*.js` are ad-hoc scripts).
- **Coverage is exact**: 38 tracked `.js/.mjs/.py/.sh` files and 38 files with nodes.
- **Two gotchas.** (1) `embeddings_count: 0` → `semantic_search_nodes_tool` returns nothing; use
  `query_graph_tool` / SQL instead. (2) Pass **`repo_root: "/appdata/paperless-ai-src"`** on every CRG
  call: Hermes launches outside this repo. It is registered with CRG (2026-10-16), but `repo_root` outranks
  the registry.
- **A directory rename invalidates every node** (`file_path` is stored absolute) and an incremental
  update then answers "No changes detected" while dead paths survive — only
  `code_review_graph build --repo <path>` (or `full_rebuild: true`) rewrites them. That is why the move
  was followed by a full rebuild. Assert it, don't assume it: 0 nodes under the old prefix, 583 here.

## Architecture read (where a change ripples)

- Communities: `js-initialize` (241, JS, cohesion 0.34), `services-document` (192, JS, 0.28),
  `paperless-ai-search` (51, Python, 0.18), `models-documents` (26, 0.00), `routes-process` (19, 0.02),
  `paperless-ai-data` (15, 0.08). Two real clusters plus a weakly-coupled Python island (`main.py`, the RAG side).
- Flows by criticality — the re-test list for any change:
  - `0.74 startServer` (server.js:628, 46 nodes / 5 files) — the whole document-intake path:
    `initializeDataDirectory` → `saveOpenApiSpec` → `startScanning` → `scanDocuments` → `scanInitial`
    → `processDocument` → `saveDocumentChanges` + the paperlessService loaders.
  - `0.69 analyzeDocument` ×3 (customService.js:30, azureService.js:31, openaiService.js:40) and
    `0.65 analyzeDocument` (ollamaService.js:78) — four near-identical provider paths. Change the shared
    prompt/restriction helpers, or `services/aiServiceFactory.js`, rather than four siblings.
  - `0.59 analyzePlayground` ×4 (same four services), `0.58 saveConfig` (setupService.js:209),
    `0.53 sendMessageStream` (chatService.js:102), `0.52 askQuestion` (ragService.js:56),
    `0.47 analyzeDocument` (manualService.js:36), `0.47 getDocumentsWithMetadata` (documentsService.js:30).
- Widest ripple (in-degree): `initialize` @ `services/paperlessService.js` (26 callers), `save_state`
  @ `main.py` (22), `toggle` @ `public/js/settings.js` (14), `normalizeArray` @ `routes/setup.js` (12),
  `processRestrictionsInPrompt` (10), `getDocument` (10), `truncateToTokenLimit` (9), `parseEnvBoolean` (9),
  `isConfigured` (8). With 0 `TESTED_BY` edges, nothing catches a regression except a live scan.

## Live deployment (re-verified 2026-10-16)

- Container `paperless-ai` on `appnet`, image **`clusterzx/paperless-ai:latest`** (digest `sha256:2b6588…`,
  no OCI revision label), healthy, `StartedAt 2026-10-15T16:40:56Z`, 0 restarts, port `8084:3000`,
  `https://paperless-ai.home` (Caddy) → 302 `/login`.
- **This repo is NOT what runs.** The container is upstream's published image; there is no local build
  pipeline in the stack. A code change here is inert until an image is built and the compose `image:` swapped.
- Peers it talks to, all up: `paperless-ngx` (:8092, `paperless.home`), `paperless-gpt` (:8088),
  `paperless-db`, `paperless-redis`, `paperless-tika`, `paperless-gotenberg`, `ollama`.
- **Config comes from two files, and compose wins.** compose `env_file: .env` (stack dir, 13 keys) is
  injected into the process, and `config/config.js:5` then does `dotenv.config({path: './data/.env'})`
  (= `/app/data/.env`, ~44 keys) **without override** — so the stack `.env` wins wherever both define a key
  and `/app/data/.env` supplies the rest (that is where the setup wizard writes).
- **Config now has one owner per layer — fixed 2026-10-16.** The stack `.env` holds deployment wiring
  only (TZ, PAPERLESS_URL/NGX_URL/HOST, RAG_SERVICE_*, MULTI_SERVICE_ENABLED, with the rule written in
  its header); everything the Settings UI writes lives in `/app/data/.env` alone. Effective config
  before vs after: **51 keys, exactly one value changed** — `OLLAMA_MODEL` `qwen3:8b` →
  `qwen2.5-7b-128k:latest`. Verified at the ollama layer with the app's own playground system prompt and
  schema: valid JSON (title/correspondent/tags/document_type/document_date/language) in 4.3 s. No model
  pull was needed. Rollback: `/root/backups/paperless-ai-{interface,data}.env_20261016_195154`.
- Other effective env values (no secrets): `AI_PROVIDER=ollama`, `OLLAMA_API_URL=http://ollama:11434`,
  `TZ=America/New_York`, `PAPERLESS_API_URL=http://paperless-ngx:8000/api`,
  `RAG_SERVICE_URL=http://paperless-ngx:8000`, `RAG_SERVICE_ENABLED=true`, `MULTI_SERVICE_ENABLED=true`.
  Both `.env` files carry `PAPERLESS_API_TOKEN`, `PAPERLESS_USERNAME`, `API_KEY` — names only, never print values.
- `SCAN_INTERVAL=*/30 * * * *` in `/app/data/.env` (not in the stack `.env`), matching the scans seen in
  the logs at :00 and :30.

## Upstream status (why this fork is a mirror)

Upstream README, top of `main`: "This repo is currently not maintained… I'm currently rewriting the
entire codebase… With the upcoming official AI integration in Paperless-ngx itself, I'm also not sure
if I'll complete the rewrite or continue maintaining this repo at all." Last commit on upstream `main`
is the same `d2fda69` of 2026-03-31 (previous: `0eee0c6` "Bump version to 3.0.9", 2025-11-04);
57 open issues; `pushed_at 2026-10-16T01:31` is traffic on other refs, not `main`. Practical
consequence: "upstream drifted" is never the explanation for something here, and a Paperless-ngx AI
feature may make this component redundant.

## Toolchain

- Node app: `server.js` (entry, Express + EJS + PM2 in the image), `services/*` (16 files),
  `routes/*`, `models/`, `public/js/*` front-end, `schemas.js`, `swagger.js`. `package.json` scripts
  are `{test: nodemon server.js}` — **there is no test runner**; `eslint.config.mjs` + `prettierrc.json`
  define style. Key deps: express, better-sqlite3 + sqlite3, openai, axios, cheerio, ejs, jsonwebtoken,
  bcryptjs, node-cron, dockerode, tiktoken, swagger-ui-express.
- Python (RAG side): `main.py` (~96 kB, FastAPI), `requirements.txt` (fastapi, uvicorn, chromadb,
  sentence-transformers, torch, rank-bm25, nltk, numpy, pydantic). `start-services.sh` starts both;
  `Dockerfile` (Node app) and `Dockerfile.rag` (Python/RAG image) are the two build targets.
- `.docling`/OCR helpers in the tree are upstream's; nothing here has been modified yet.

## Decisions

- 2026-10-16 — moved the checkout out of the container's data bind mount to `/appdata/paperless-ai-src`;
  plain `mv` (same filesystem), verified git intact, container healthy, `paperless-ai.home` still 302.
- 2026-10-16 — full graph rebuild at the new path (a rename leaves dead absolute paths that no
  incremental fixes); asserted 0 nodes at the old prefix / 583 at the new.
- 2026-10-16 — registered the repo with CRG (`/appdata/paperless-ai-src`), so cwd auto-detect and
  cross-repo search can find it.
- 2026-10-16 — committed this `MEMORY.md` to the fork rather than gitignoring it, following the
  measured convention of this homelab's other forks.
- 2026-10-16 — the stack `.env` was split by role (deployment wiring only) and the app-owned
  `/app/data/.env` was aligned, so the Settings UI's saved values are the effective ones; the model
  moved to `qwen2.5-7b-128k:latest` (51 keys, one value changed). Backups in `/root/backups/`.
- Not chosen: writing a `FORK.md` — there is no fork delta to explain yet (the file is expected once
  this fork carries its own changes).

## Open questions

1. What is this checkout for? Nothing fork-specific has been changed yet, so either a local patch set
   (e.g. fixing the model config, or a feature Paperless-ngx's own AI won't cover) or a local image build.
2. Do we build and run our own image (`Dockerfile` here) instead of `clusterzx/paperless-ai:latest`? That
   is the only way a change in this repo reaches `paperless-ai.home`.
3. ~~Fix the `qwen3:8b` vs available-models mismatch~~ — settled 2026-10-16: the stack `.env` no longer
   carries an app setting, and the effective model is `qwen2.5-7b-128k:latest`, proven to serve at the
   ollama layer.
4. The two `.env` files no longer overlap on app settings (stack = wiring, app file = settings). What
   remains open is the app-side hardening: a startup conflict report and a model-availability check —
   planned at `/appdata/A--docker_stacks/.hermes/plans/2026-10-16_195403-paperless-ai-config-precedence-and-model-check.md`,
   awaiting Ron's decisions before any code.

## Checks worth running

```sh
# graph fresh for this checkout?
cd /appdata/paperless-ai-src && bash /root/.hermes/scripts/check-crg.sh    # WARN about "not installed" = known PATH artifact
# container + route
docker inspect paperless-ai --format '{{.State.Health.Status}} since {{.State.StartedAt}}'
curl -s -o /dev/null -w '%{http_code} -> %{redirect_url}\n' -H 'Host: paperless-ai.home' http://10.1.10.10:8084/
# does ollama serve the configured model?
docker exec paperless-ai node -e "fetch('http://ollama:11434/api/tags').then(r=>r.json()).then(d=>console.log(process.env.OLLAMA_MODEL, d.models.map(m=>m.name)))"
```

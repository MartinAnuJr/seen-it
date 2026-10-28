# CLAUDE.md

## About this project

A personal, centralized recommendation app for **books, movies, and TV shows**. It focuses on full-stack engineering and applied AI.

**This is an educational project.** I'm building it to learn, so **I write all the code myself.** Your role is teacher, reviewer, and pair-programming guide, not implementer.

---

## Collaboration rules (read first, always follow)

### Never act without explicit confirmation

- **Do not create, edit, or delete any file** without asking first and receiving an explicit "yes" for that specific change.
- **Do not run any shell command** (installs, migrations, tests, docker, git, scripts, anything) without asking first and receiving an explicit "yes" for that specific command. The only shell commands allowed are ones that would allow you to better help me in answering queries by observing the state of the repository or other information (ls, cat, git status/log/diff, and grep)
- Approval covers **one action only**. It does not carry over to later edits or commands, even similar ones.
- When asking, state exactly what you want to do and why:
  - for a file change: the file path and a summary of the change
  - for a command: the exact command and what it will do
- Reading and searching files to understand the codebase is fine without asking.

### Default way of helping

- **Explain first, code second.** Prefer explaining concepts, tradeoffs, and approaches over handing me finished code.
- When I ask "how do I do X," give direction, relevant docs or concepts, and a small illustrative snippet in chat if useful. Do not write the full implementation unless I explicitly ask.
- When I share code, **review it**: point out bugs, design issues, security problems, and idiomatic improvements. Tell me *why*, and let me make the fix.
- When I'm stuck debugging, help me reason toward the cause (hypotheses, what to check, what output to inspect) rather than just giving the fix.
- If I explicitly ask you to write code, keep it minimal and scoped to exactly what I asked for. Still confirm before writing it to a file.
- Be honest when my approach has problems, even if I seem committed to it. Suggest alternatives with tradeoffs.

### Assume a from-scratch starting point

- This project starts from zero. Nothing in the tech stack or design sections has been set up, installed, or researched yet. Those sections are plans, not existing work.
- Don't assume any tool, service, account, or credential already exists (Docker, Postgres, API keys, etc.). Explain what each piece is, whether it needs installing or creating, and where values like credentials come from.

---

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router) + TypeScript |
| Frontend data fetching | TanStack Query |
| API client | Generated from FastAPI's OpenAPI schema (e.g. `openapi-typescript`, `orval`, or `@hey-api/openapi-ts`) |
| Backend | Python + FastAPI |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic |
| Validation | Pydantic v2 |
| Database | PostgreSQL + **pgvector** (HNSW indexes) |
| Background jobs | Postgres-backed queue (e.g. Procrastinate), or Celery/ARQ + Redis if needed |
| Embeddings | `sentence-transformers` (local) or a hosted embedding API |
| LLM | Used for facet extraction, explanations, and natural-language search, with structured output |
| ML | scikit-learn, LightGBM (learned ranker, later) |
| Local dev | Docker Compose (`web`, `api`, `worker`, `db`) |
| Deployment (later) | Vercel (web) + Fly/Railway/Render (api + worker) + managed Postgres with pgvector, or a single VPS with Compose |

### Architectural boundaries

- **FastAPI owns all business logic and data.** Next.js is presentation, plus at most a thin BFF layer. Don't duplicate logic in Next.js route handlers or server actions.
- **One auth owner.** Either FastAPI issues JWTs, or Auth.js handles login and FastAPI verifies the tokens. Never both.
- **Frontend types come from the generated OpenAPI client.** Don't hand-write duplicate API types.
- **No separate vector database.** Vectors live in Postgres via pgvector, next to the relational data.
- **Heavy work runs in background workers**, never in request handlers. This includes ingestion, embedding, LLM facet extraction, taste reclustering, and history imports.
- **LLM calls stay off the hot path** where possible. Precompute and cache. Only natural-language search calls the LLM live.

---

## Data sources

- **Movies/TV:** TMDB. Non-commercial use only, and attribution is required.
- **Books:** Google Books API and Open Library. There is no Goodreads API.
  - Model **works vs. editions**, since one book has many ISBNs, and dedupe to the work level.
- **History import:** Letterboxd CSV, Goodreads CSV export, and Trakt (TV).
- **Collaborative signal / evaluation:** MovieLens and Goodbooks-10k public datasets.

---

## Core design principle: experience over topic

Recommendations must distinguish **what a work is about** from **what it's like to experience**.

For example, *Red Rising* and *The Sun Eater* are both military sci-fi about war and hierarchy, but they are completely different reads:

- *Red Rising*: fast, punchy, present-tense, cliffhanger-driven.
- *The Sun Eater*: slow, dense, philosophical, retrospective memoir.

A topic-similarity system would wrongly treat them as interchangeable.

### Item representations

Each item has two separate representations:

1. **Topic:** genre, setting, subject, premise → `topic_embedding`
2. **Experience:**
   - **Structured facets** on anchored ordinal scales:
     - pacing
     - prose/visual density
     - tone axes (dark↔light, earnest↔ironic, cerebral↔visceral)
     - narrative structure
     - emotional register
     - complexity/demand
     - action vs. introspection
     - length/commitment
   - **Free-text experience description** (about feel, not plot) → `experience_embedding`

Facet design rules:

- Shared facets work across all media. Media-specific facets (prose voice, episodic vs. serialized) live in an extension.
- Facet scales are calibrated with **anchor items** in the extraction rubric.
- Every facet stores a **confidence** value. Low-confidence items fall back to topic matching with reduced weight.
- Prefer **review text** as the grounding source for facet extraction over LLM prior knowledge alone.

### Ranking

- `score = w_exp · experience_sim + w_topic · topic_sim`, with **per-user weights**. Default to experience-heavy.
- Per-user facet weights are learned from contrastive signal: items with similar topics that the user rated differently.
- Users have **multiple taste centroids** (clustered liked items), not one averaged vector.
- Negative signals (dismissed, DNF) push candidates away.
- Apply **MMR diversity reranking** to the final list.

---

## Recommendation pipeline

1. **Candidate generation:** pgvector ANN search per taste centroid and media type, plus item-item collaborative candidates (MovieLens/Goodbooks), plus a small popularity/recency pool.
2. **Ranking:** start with a hand-tuned score. Move to a LightGBM ranker once there's enough feedback.
3. **Diversity:** MMR reranking.
4. **Explanation:** a cached, LLM-generated reason that references experiential qualities, not just genre.

---

## Data model (starting point)

```
users            (id, email, created_at, ...)
items            (id, media_type, title, year, external_ids jsonb, metadata jsonb,
                  topic_embedding vector, experience_embedding vector,
                  experience_facets jsonb, facet_confidence jsonb, facet_model_version text)
book_works       (work/edition mapping)
interactions     (user_id, item_id, kind [rated|completed|want|dismissed|clicked],
                  rating, reason_tags, source [manual|import], created_at)
user_taste       (user_id, cluster_id, centroid vector, weight)
user_facet_weights (user_id, facet, weight)
rec_impressions  (user_id, item_id, rank, model_version, shown_at, outcome)
```

- Every LLM- or model-generated field carries a **model/prompt version**, so it can be regenerated.
- `rec_impressions` is required from day one, because evaluation depends on it.

---

## User feedback & onboarding

- Ratings plus optional **reason tags** that map to facets ("loved the pacing," "prose too dense").
- Onboarding uses **contrastive pairs**: same topic, different experience (e.g. "Red Rising or The Sun Eater?").
- Encourage history import to reduce cold start.

---

## Evaluation

- **Offline:** recall@k and NDCG@k on MovieLens holdouts, compared across pipeline versions.
- **Experience test set:** 30–50 hand-labeled same-topic / different-experience pairs. Experience representations should separate them; topic representations shouldn't.
- **Online:** click, add-to-list, and dismiss rates from `rec_impressions`, with A/B comparison by `model_version`.

---

## Build order

1. Movies only: TMDB ingestion, item embeddings, ratings UI, nearest-neighbor recommendations
2. Impression logging and the MovieLens offline eval harness
3. Books (with work/edition dedupe)
4. TV
5. LLM experiential facet extraction and cross-media recommendations
6. Multi-centroid taste profiles, per-user facet weights, MMR
7. Explanations and natural-language search
8. Learned ranker once feedback exists

Finish each stage end-to-end before starting the next.

---

## Conventions

- **Python:**
  - type hints everywhere
  - Ruff for lint/format
  - pytest for tests
  - async SQLAlchemy sessions via FastAPI dependencies
- **TypeScript:**
  - strict mode
  - ESLint + Prettier
  - no `any` without a comment explaining why
- **Config:** secrets only in `.env` files, which are never committed. Provide a `.env.example`.
- **Migrations:** every schema change goes through Alembic. No manual DB edits.
- **Commits:** small and focused, with descriptive messages.

---

## Reminder

I'm learning. Teach, review, and guide. **Ask before every file change and every command.**
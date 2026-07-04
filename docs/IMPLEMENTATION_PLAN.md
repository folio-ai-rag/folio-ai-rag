# Implementation Plan — Feature-by-feature roadmap

Purpose
-------
This technical document translates the SDD (Folio_SDD.md) into a prioritized, actionable sequence of small, reviewable work items. Each work item maps to SDD sections, has acceptance criteria, testing guidance, and suggested PR size so contributors (including newcomers) can implement the product feature-by-feature.

How to use
----------
- Read the SDD first to understand the rationale and constraints.
- Pick the next item in the Phase 1 list (MVP). Each item is small enough for a single PR.
- Link the PR to the SDD section(s) and to the item below.
- Follow the PR checklist: tests, migration notes, SDD link, manual test steps.

Phases and high-level ordering
------------------------------
We implement in phases that mirror the SDD. Each phase is a vertical slice: user flow from API → worker → data store → UI (or CLI test harness). Phase 1 is the MVP: prove the core loop (upload → ingest → cited QA) for a single document and user.

Phase 1 — MVP (core plan, implement in this order)
--------------------------------------------------
Implement the smallest end-to-end vertical slices first so you can iterate on retrieval quality quickly.

1) Developer infra & CI (pre-work)
   - Goal: reproducible dev environment and CI that runs linters and unit tests.
   - Tasks:
     - Add Docker compose or dev scripts for Postgres+pgvector and Redis (dev-only).
     - Add a GitHub Actions workflow that runs ruff/black and pytest unit tests.
   - Why first: catches environment drift and keeps PRs fast to validate.
   - Acceptance: CI runs and passes on a trivial commit.
   - Estimated: 1–2 days.

2) Database schema & migrations
   - Goal: core tables exist: organizations, users, documents, document_pages, chunks, embedding_refs, messages, jobs.
   - Tasks:
     - Create Alembic baseline migration matching SDD (subset sufficient for Phase 1).
     - Add scripts to create a local dev DB and apply migrations.
   - Tests: integration test that runs migrations against a fresh Postgres container and validates critical tables exist.
   - Estimated: 0.5–1 day.

3) Auth skeleton
   - Goal: email/password sign-up and login endpoints (JWT + refresh cookie) working for local dev.
   - Tasks:
     - Implement POST /auth/register, POST /auth/login, POST /auth/refresh, GET /auth/me.
     - Integrate Argon2id password hashing (library) and simple JWT issuance (short TTL).
   - Tests: unit tests for hashing/verify and auth endpoints (integration with DB).
   - Acceptance: a user can register and login locally.
   - Estimated: 1–2 days.

4) Upload-init / upload-complete API + direct-to-S3 design (local stub)
   - Goal: implement the API contract for initiating an upload and completing it. For Phase 1 dev, S3 can be stubbed or use MinIO/localstack.
   - Tasks:
     - Implement POST /documents/upload-init to create a documents row and return a pre-signed URL set (or a local stub that accepts parts).
     - Implement POST /documents/upload-complete that validates parts and sets document status to UPLOADED and enqueues the first job (virus-scan).
   - Tests: API unit tests that simulate upload-init/complete flow; integration using local S3 emulator.
   - Acceptance: the API returns an upload token and the complete endpoint transitions to UPLOADED and enqueues a job row.
   - Estimated: 1–2 days.

5) Job queue & worker skeleton (Celery + Redis)
   - Goal: workers and queues exist and can run simple jobs end-to-end.
   - Tasks:
     - Add Celery app and a `jobs` table to track job state.
     - Implement a simple worker that picks up a document and emits job events to Redis pub/sub (or DB updates).
   - Tests: unit tests for job lifecycle; simple end-to-end test that enqueues a virus-scan job and the worker marks it succeeded.
   - Acceptance: job transitions appear in DB and are visible via a GET /admin/jobs endpoint.
   - Estimated: 1 day.

6) Virus-scan worker (sandbox stub)
   - Goal: safe path for files — for dev, simulate ClamAV run and transition either to REJECTED or to next stage.
   - Tasks:
     - Implement virus-scan job that runs in a no-egress container in prod; in dev stub run a deterministic check (e.g., check for a .infected flag file in S3 key).
     - On success, enqueue extraction job.
   - Tests: unit and integration tests verifying state transitions.
   - Acceptance: uploaded document moves from UPLOADED → SCANNING → EXTRACTING (enqueued)
   - Estimated: 0.5 day.

7) Extraction — fast digital path (PyMuPDF)
   - Goal: extract text for the large majority of pages cheaply.
   - Tasks:
     - Implement page-level classifier stub that routes all pages to FAST_DIGITAL initially.
     - Implement pymupdf_extract(page) to produce plain text and simple metadata (image_coverage, text_ops count).
     - Persist document_pages rows with raw_text_s3_key (or inline for dev) and has_table flag false.
   - Tests: unit tests on small PDF fixtures; integration test on a multi-page PDF verifying page_count and stored text.
   - Acceptance: document_pages created for every page with extraction_confidence and raw text written to S3/local fixture.
   - Estimated: 2–3 days.

8) Chunking service — baseline recursive chunker
   - Goal: implement structure-aware recursive chunking per SDD with overlap and context prefix.
   - Tasks:
     - Implement recursive_split + apply_sliding_overlap functions.
     - Produce chunk rows with content, token_count, content_hash.
   - Tests: golden chunking tests for a fixture PDF (check chunk boundaries stable).
   - Acceptance: chunks exist and map to page ranges; golden test passes.
   - Estimated: 2 days.

9) Embedding pipeline (provider stub) + embedding cache
   - Goal: implement batched embedding calls (initially stubbed or using a small, cheap model) and Redis cache lookup by content_hash.
   - Tasks:
     - Implement EmbeddingProvider interface and a local-inference or mock provider.
     - Batch chunks (~100) per call and write embedding_refs rows with `vector` column for pgvector.
   - Tests: unit tests for cache lookup and integration test writing embeddings to Postgres.
   - Acceptance: embedding_refs rows created for all chunks with vector data.
   - Estimated: 2 days.

10) Vector store (pgvector) index & lexical tsvector
    - Goal: configure pgvector extension and GIN index on chunk content tsvector to enable hybrid retrieval later.
    - Tasks:
      - Add migration to create extension and indexes; configure HNSW params conservatively for dev.
    - Tests: integration ensuring nearest-neighbor query returns expected rows on seeded vectors.
    - Estimated: 1 day.

11) Summary job (map-reduce minimal)
    - Goal: produce a TL;DR and executive summary using the reduce-stage approach but on a single cheap model call for Phase 1.
    - Tasks:
      - Implement map stage: per-section summarization (for Phase 1, sections = page ranges or doc-level fallback).
      - Implement reduce stage into a single summary row.
    - Tests: unit tests asserting summaries are produced and persisted.
    - Acceptance: GET /documents/{id}/summary returns TL;DR once document status = READY.
    - Estimated: 1–2 days.

12) Retriever + conversation skeleton (dense-only + lexical fuse later)
    - Goal: a working chat endpoint that embeds the user query, finds nearest chunks, and streams a model answer with citation markers.
    - Tasks:
      - Implement POST /conversations/{id}/messages streaming endpoint (SSE or a test harness that prints tokens).
      - Implement query embedding (same model family as doc embeddings) and a simple nearest-neighbor lookup over pgvector (top 10), plus tsvector lexical candidate (top 10) merged with simple RRF or rank merge.
      - Implement a simple Prompt Builder that inserts top 5 chunks with markers [C1]... and calls the LLM Gateway (initially mocked or using a dev model API) to generate an answer.
      - Implement Citation Mapper that maps markers to chunk.page_start.
    - Tests: integration test that seeds a document with a known answer and asserts the returned message contains a citation with the expected page number.
    - Acceptance: a user can ask a question and receive a streamed answer with at least one clickable citation that maps to the page in the document viewer.
    - Estimated: 3–5 days (complex part of Phase 1).

13) End-to-end validation & QA
    - Goal: fix gaps, add missing tests, and ensure the p95 ingestion and chat latency targets are plausible in staging.
    - Tasks:
      - Run Playwright end-to-end test that performs upload → wait for ready → ask a golden question → validate citation jump.
      - Add load tests (k6) for a few concurrent ingestions and chat turns.
    - Acceptance: staging run completes within acceptable time for small test corpus.
    - Estimated: 2–4 days.

Phase 1 wrap: polish, docs, and release candidate
-------------------------------------------------
- Finalize README and CONTRIBUTING with the actual commands you used.
- Add golden chunking/retrieval fixtures to CI as regression tests.
- Tag a v0.1.0 and deploy to a small staging environment.
- Monitor metrics and iterate.

Phase 2 — Growth (high-level)
------------------------------
After the single-document vertical slice is stable, implement Phase 2 features in this order:
1. Hybrid retrieval (dense + sparse) and cross-encoder re-ranking (re-ranker integration) — SDD Section 14.
2. Multi-document chat (deal-room) — modify schema and retriever to accept document sets.
3. UI features: bookmarks, highlights, share links, exports.
4. Section summaries, mind maps, custom-prompt summaries.
5. Email/in-app notifications, admin dashboard.

Each item above should be broken down like Phase 1 into PR-sized tasks: migration, API, workers, tests.

Phase 3 — Scale & monetization (high-level)
--------------------------------------------
1. Billing (Stripe) and plan enforcement.
2. Team workspaces, SSO (Microsoft Entra), and audit logging improvements.
3. Kafka migration for queues, Qdrant evaluation and migration runway.
4. SOC 2 readiness work and operational runbooks.

PR structure and checklist (enforced on every PR)
-------------------------------------------------
- Title uses Conventional Commits.
- Link to SDD section(s) in PR body.
- Short description of change and rationale.
- Testing steps + automated tests included.
- If DB changes: include Alembic migration and manual rollback notes.
- Security considerations called out (uploads, secrets, BYO keys).
- CI green and at least one reviewer approval.

PR-sizing guidance
-------------------
- A PR should be reviewable in ~30 minutes (diff < 400 lines ideally).
- If a task is large, split it into:
  1) Schema & API skeleton (no heavy logic),
  2) Worker/implementation, and
  3) Tests & polish.

Example mapping: "POST /documents/upload-init" implementation
----------------------------------------------------------------
- PR A: add DB migration for `documents` table and create upload-init route skeleton (no S3 yet). Include unit tests for DTO validation.
- PR B: implement S3 pre-signed URL logic and client example; add integration test with localstack/minio in CI.
- PR C: implement upload-complete verification and enqueue initial job; add job table migration and integration test for job enqueue.

Developer tips (for beginners)
------------------------------
- Keep changes small and iterate.
- Write tests before or alongside implementation for critical logic (chunking, extraction boundaries).
- Use the SDD as the authoritative design; reference it in every PR.
- Prefer safe defaults: abort/return clear errors on malformed PDFs rather than retries.

Next actions you want me to do
-----------------------------
I can:
- Commit these two files into the repository now (README.md + docs/IMPLEMENTATION_PLAN.md).
- Create the minimal dev Docker compose and a CI workflow next.
- Draft Alembic migration templates for the Phase 1 schema.

Which of these do you want me to run next?
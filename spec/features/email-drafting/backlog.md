## backlog.md

Backlog — Email Drafting (v1 and beyond)

Purpose
Track post-MVP work, experiments, and tech debt. Priorities: P0 (blocker/urgent), P1 (next up), P2 (nice-to-have), P3 (later/idea). Effort: S (≤0.5d), M (1–2d), L (3–5d).

========================================
MVP Hardening (P0–P1)

[P0][S] Add unique DB constraint for idempotency
– UNIQUE (message_id, internet_message_id) on email_events; ensure graceful conflict handling.

[P0][S] HTML sanitizer whitelist
– Explicit allowlist: p, ul, ol, li, a, strong, em, br; strip javascript: URLs and on* attributes.

[P0][S] Redaction utilities
– Regex for phones and student emails (mask local part); unit tests.

[P1][M] Low-confidence workflow
– Flag records < 0.55 and surface them in reviewer list (even before we have a UI).

[P1][M] Logging normalization
– Structured JSON logs with request_id/message_id; log retries/backoff.

[P1][S] Health/readiness endpoints
– /health returns DB + models; add /ready that checks DB connectivity.

========================================
Review Experience (P1)

[P1][M] Minimal reviewer page (read-only)
– Table: subject, bucket, confidence, received_at, status.
– Detail view: original body (sanitized), retrieved passages, rendered draft, citations, link “Open in Outlook”.

[P1][S] “Mark as reviewed” toggle in DB
– Status field or separate table; no Outlook writeback needed.

[P1][S] Manual re-classify + re-draft
– Two buttons (or CLI endpoints) to re-run classification or draft with override bucket.

========================================
Retrieval & Quality (P1–P2)

[P1][M] Weekly re-ingest job
– Cron or launchd to rebuild embeddings; diff detection by file hash.

[P1][S] Query formulation improvements
– Subject weighting, body truncation, de-dup of passages; test impact on similarity.

[P2][M] Reranker
– Local cross-encoder rerank (e.g., bge-reranker base via Ollama or CPU), top-k→top-3.

[P2][S] Domain synonyms
– Expand keywords (“W-9”→“W9”, “honorarium”→“speaker fee”), central list.

========================================
Classification (P1–P2)

[P1][S] Regex rules audit
– Add tests per bucket; expand patterns for false negatives found in the field.

[P1][S] LLM fallback JSON guard
– Strict JSON parse with schema; on failure, default to Misc + Next steps.

[P2][M] Few-shot examples
– Add 2–3 labeled examples per bucket to the fallback prompt; measure accuracy lift.

========================================
Outlook/Graph Integrations (P1–P2)

[P1][S] Skip already tagged items
– Filter messages where category includes “Auto-Drafted”.

[P1][S] Robust checkpointing
– Store checkpoint by datetime + ETag; handle clock skew.

[P2][M] Attachments mention helper
– In draft: “Attachments to include: …” derived from policy metadata (still no auto-attach).

[P2][M] Sent-folder correlation (manual audit)
– After you send a reply, optionally log the final sent message id to link to the draft source.

========================================
Security & Compliance (P1–P2)

[P1][S] Token persistence hardening
– Use macOS Keychain; rotate refresh tokens if available.

[P1][S] Secrets scan pre-commit
– Add pre-commit hook for trufflehog/gitleaks (local only).

[P2][M] Access control for reviewer page
– Local HTTP basic auth or OS account check.

========================================
Ops & Tooling (P1–P2)

[P1][S] Makefile targets
– make rag, make worker, make ingest FILE=…, make test.

[P1][S] .env validation
– Fail fast with missing critical vars; print models and DB at startup.

[P2][M] Lightweight metrics
– Export basic counters (processed, errors, latency) to a local endpoint; optional Prometheus.

[P2][S] Daily DB backup
– Local dump + rotation; retention 7–14 days.

========================================
Backlog Features (Post-MVP P2–P3)

[P2][M] Multi-mailbox support
– Config to process a second mailbox (shared mailbox later).

[P2][M] Policy diff awareness
– Notify when a doc changed and re-ingest happened; annotate drafts with version.

[P2][M] Nightly “polish” pass
– Re-run LLM with a larger model overnight for low-confidence items; produce improved drafts for next-day review.

[P3][L] Attachment automation
– Auto-attach canonical forms (W-9, CME) from a safe local directory; strict filename allowlist.

[P3][L] Non-English support
– Add translation layer + bilingual prompts.

[P3][L] Webhooks / Push subscriptions
– Replace polling with Graph subscriptions + delta queries.

========================================
Research Spikes (Time-boxed)

[Spike][S] Alternative embeddings
– Compare nomic-embed-text vs bge-small for speed/quality on your corpus.

[Spike][S] Model sizing
– Llama 3.1 8B vs 13B on drafting quality and latency on your hardware.

[Spike][S] Reranker feasibility via CPU
– Check latency using a small cross-encoder without GPU.

========================================
Tech Debt / Cleanup

[TD][S] Normalize timestamps to UTC everywhere (DB and logs).
[TD][S] Centralize HTML templates for drafts; one place to adjust tone/format.
[TD][M] Alembic migrations for schema versioning.
[TD][S] Replace ad-hoc regexes with compiled patterns + tests.
[TD][S] Add typed DTOs for API contracts to prevent shape drift.

========================================
Exit Criteria (Backlog tranche complete)

– Reviewer page usable for daily workflow.
– Weekly re-ingest automated.
– Low-confidence items easy to find and improve.
– Logs consistently redacted and structured.

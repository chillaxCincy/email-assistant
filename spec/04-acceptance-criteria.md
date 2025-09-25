## 04-acceptance-criteria.md

Acceptance Criteria — Email Drafting (v1)

Core Functional
	•	Worker polls Outlook inbox every 180s (configurable via env).
	•	Each new, non-draft email is classified into exactly one of 7 buckets.
	•	Rule-based classification applies when keyword matches are clear; otherwise LLM fallback decides.
	•	RAG service retrieves ≥2 relevant passages for typical queries (when available).
	•	Drafts are generated in HTML with consistent structure: greeting, answer, attachments list (if any), citations line.
	•	Every draft ends with “Source policy: [Doc:Section; …]”.
	•	If retrieval weak (<0.55 confidence), draft includes explicit “Next steps” request.
	•	Drafts are saved in Outlook Drafts folder, never sent.
	•	Original messages are tagged with categories: Auto-Drafted + the assigned bucket.
	•	All processing results (classification, draft, citations, confidence) logged to Postgres.

Quality / Nonfunctional
	•	Draft available within 5 minutes of message arrival (poll-based latency).
	•	RAG search completes in ≤1s typical; draft generation ≤60s typical.
	•	No duplicate processing: each message idempotent by internetMessageId.
	•	Errors trigger retries with exponential backoff (1m → 4m → 15m → 60m, max 5).
	•	On retry exhaustion, status set to error; no half-written drafts.
	•	Logs redact PHI: student email local parts, phone numbers.
	•	Logs are structured JSON with timestamps, status transitions, and redaction applied.
	•	HTML sanitizer strips scripts, javascript: links, on* attributes.
	•	All configuration lives in .env, secrets never committed.
	•	Health endpoint reports DB + model readiness.
	• Retry loop for Graph/LLM errors must stop after max 5 attempts (no infinite retries).

Safety
	•	Codebase contains no call to Graph /send.
	•	No attachments auto-included in replies (MVP).
	•	No exposure of public webhooks.
	•	No processing of shared or secondary mailboxes (MVP).

Success Criteria (release gate)
	•	≥90% of sampled test emails (TC-01..06) produce usable drafts with ≥2 citations.
	•	TC-07 or TC-09 demonstrates correct “Next steps” behavior.
	•	TC-10 (HTML safety) passes.
	•	TC-11 (redaction) passes.
	•	TC-12 (idempotency) passes.
	•	TC-13 (retry) either succeeds after retry or fails cleanly with status=error and no draft.
	•	Zero /send events in logs or audit.

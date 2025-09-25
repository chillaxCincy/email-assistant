Nonfunctional Specification — Email Drafting (v1)

Performance
	•	Throughput: System must handle ~200 emails/day (≈10/hour peak).
	•	Latency: Draft available within 5 minutes of email arrival (polling-based).
	•	RAG search: ≤1 second per query on Mac Studio (Postgres + pgvector).
	•	LLM draft generation: ≤60 seconds per email, with retries.

Reliability
	•	Idempotency: Each email processed once only, keyed by internetMessageId.
	•	Retries: Exponential backoff for Graph API and LLM errors (1m → 4m → 15m → 60m).
	•	Error handling: If retries exhausted, mark status=error; never leave emails half-processed.
	•	Persistence: All events, classifications, drafts, and errors logged in Postgres.

Security
	•	Data scope: Process only the inbox of a single delegated user (MVP).
	•	Secrets management: Configured via .env, never committed.
	•	Encryption: OAuth tokens stored securely (macOS Keychain preferred).
	•	PHI/PII:
	•	Redact student full names, phone numbers, student emails (local part) in logs.
	•	Do not persist attachments by default.
	•	HTML safety: Strip scripts/unsafe attributes from generated drafts; UTF-8 enforced.

Maintainability
	•	Code structure: Two services (rag_api and graph_worker), isolated responsibilities.
	•	Config: All tunables (poll interval, top_k, thresholds) via env vars.
	•	Docs: Each feature has versioned specs (problem, functional, nonfunctional, api-contracts, test-cases).
	•	Schema evolution: Use migrations (e.g., Alembic) for Postgres updates.
	•	Logging: Structured JSON logs with timestamps, status transitions, and redaction.

Scalability (future-proofing)
	•	Storage: Policy docs < 50 MB; embeddings < 1M chunks (well within Postgres + pgvector).
	•	Compute: Ollama models 8B–13B run locally; larger models possible on Mac Studio with 128 GB+ RAM.
	•	Extensibility: Worker designed to support multiple categories/mailboxes in backlog; RAG service stateless.

Usability
	•	Draft output: HTML with consistent greeting, structure, citations.
	•	Review: Human approves all drafts in Outlook.
	•	Traceability: Every draft linked to original email, citations, and confidence score.

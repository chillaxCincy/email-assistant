Email Drafting Assistant (MVP)

This project is an MVP system for auto-drafting replies to student and faculty policy emails using:
	•	Rule-based classification for clear cases
	•	RAG (Retrieval-Augmented Generation) for nuanced cases
	•	PostgreSQL + pgvector for unified storage of policies, embeddings, and logs

The system is designed for self-hosted use (Mac Studio, Postgres 16, Ollama/Claude/ChatGPT backends) with safety constraints:
	•	No automatic sending (drafts only)
	•	PHI/PII redaction in logs
	•	Explicit retry limits on Graph API calls
	•	Sanitized HTML outputs

⸻

Repository Structure

.
├── adr/                # Architecture Decision Records (storage, schema, etc.)
├── docs/               # Getting started, ops, security notes
├── fixtures/           # Sample policies & emails for tests
├── services/
│   ├── rag_api/        # FastAPI service for retrieval + draft generation
│   ├── graph_worker/   # Daemon polling Outlook Graph API, writing drafts
│   └── review_app/     # (MVP placeholder) manual review UI
├── sql/                # Schema + index DDL (from ADR-0002)
├── scripts/            # DB init/migrate/reset, safety checks
└── tests/              # Test cases (see test-cases.md)

⸻

Key Specs
	•	adr-0001-storage.md: Storage choice (Postgres + pgvector)
	•	adr-0002-schema.md: Database schema (policy_docs, policy_chunks, email_events)
	•	test-cases.md: End-to-end functional tests

⸻

Quick Start
	1.	Install dependencies
	•	Python 3.11+
	•	PostgreSQL 16 + pgvector
	•	Ollama or another LLM backend
	2.	Setup environment
cp .env.example .env
make venv deps
	3.	Init database
make db-init
make migrate
	4.	Run services
make rag      # Start FastAPI
make worker   # Start Outlook worker

⸻

Safety Notes
	•	Drafts are written back as Outlook drafts only (never sent).
	•	Logs are redacted to avoid PHI/PII leakage.
	•	CI runs make no-send-check to block accidental /send calls.

⸻

Development
	•	Pre-commit hooks: linting (ruff), formatting (black), safety checks
	•	DB migrations: Alembic
	•	Tests: see tests/test-cases.md (fixtures under fixtures/)

⸻

## License
TBD – not yet chosen.

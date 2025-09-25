##adr-0002-schema.md

ADR-0002: Database Schema (MVP)

Status: Accepted (2025-09-25)

Context

Multiple specs reference tables (email_events, policy_docs, policy_chunks) but no single schema document existed. This ADR freezes MVP table names, key fields, and indexing strategy so the worker, RAG API, and review app stay consistent.

Decision

Use a simple relational schema in PostgreSQL 16 with pgvector for embeddings.

Table: policy_docs
	•	doc_id (PK, text) — stable identifier (e.g., handbook_v2)
	•	name (text) — human title (e.g., “Neurology Rotation Handbook”)
	•	version (text) — doc’s visible version (e.g., v2.1)
	•	content_sha256 (text) — hash for change detection / re-ingest
	•	source_path (text) — local path/URI for provenance
	•	last_ingested_at (timestamptz, UTC)
	•	created_at (timestamptz DEFAULT now(), UTC)

Table: policy_chunks
	•	chunk_id (PK, bigserial)
	•	doc_id (FK → policy_docs.doc_id)
	•	section (text) — logical section/heading
	•	content (text) — raw chunk text
	•	embedding (vector(768)) — nomic-embed-text
	•	page (int, nullable) — page number if known
	•	chunk_index (int) — order within the doc
	•	token_count (int, nullable) — optional for tuning
	•	created_at (timestamptz DEFAULT now(), UTC)

Table: email_events
	•	event_id (PK, bigserial)
	•	message_id (text, nullable) — Graph messageId
	•	internet_message_id (text, nullable) — RFC822 Message-Id
	•	from_addr (text) — redacted (e.g., j***@student.university.edu)
	•	from_domain (text) — domain only; helpful without PHI
	•	subject (text)
	•	body_plain_redacted (text) — sanitized/redacted copy for logs
	•	received_at (timestamptz, UTC) — from Graph
	•	bucket (text) — one of 7 categories
	•	method (text) — rule|llm (validated by CHECK)
	•	top1_similarity (float) — best retrieved passage similarity
	•	draft_self_score (float) — model’s self-score for the draft
	•	confidence (float) — min(top1_similarity, draft_self_score)
	•	citations (jsonb) — list of {doc_id, section, score}
	•	draft_message_id (text, nullable) — Outlook Draft item id
	•	categories_applied (text[] DEFAULT ‘{}’) — Outlook categories written back
	•	retry_count (int DEFAULT 0)
	•	last_error (text, nullable)
	•	status (text) — queued|processing|drafted|error (validated by CHECK)
	•	created_at (timestamptz DEFAULT now(), UTC)
	•	updated_at (timestamptz DEFAULT now(), UTC)

Constraints & Indexing

Uniqueness / Idempotency
	•	Ensure each message is processed once even if one id is missing.
	•	Unique composite on (COALESCE(message_id, ‘’), COALESCE(internet_message_id, ‘’)).
(Alternative: two partial unique indexes on message_id and internet_message_id separately.)

Checks (lightweight enums)
	•	method ∈ {rule, llm}
	•	status ∈ {queued, processing, drafted, error}

Performance
	•	email_events(received_at DESC) — recent-first queries
	•	email_events(status) — review screens / queues
	•	policy_chunks(embedding) — IVFFlat for cosine similarity (pgvector)
	•	Optional: policy_chunks(doc_id, chunk_index) — section reassembly

Consequences
	•	Confidence is explainable (we store both inputs: top1_similarity and draft_self_score).
	•	Redaction is first-class (from_addr redacted; from_domain retained for signal).
	•	Operations are simple (one DB to back up); indexes cover common access paths.
	•	Future migrations can extend this via Alembic.

Appendix: Suggested DDL (reference)

-- Extensions
CREATE EXTENSION IF NOT EXISTS vector;

-- policy_docs
CREATE TABLE IF NOT EXISTS policy_docs (
doc_id           text PRIMARY KEY,
name             text NOT NULL,
version          text,
content_sha256   text,
source_path      text,
last_ingested_at timestamptz,
created_at       timestamptz DEFAULT now()
);

--policy_chunks
CREATE TABLE IF NOT EXISTS policy_chunks (
chunk_id     bigserial PRIMARY KEY,
doc_id       text REFERENCES policy_docs(doc_id) ON DELETE CASCADE,
section      text,
content      text NOT NULL,
embedding    vector(768) NOT NULL,
page         int,
chunk_index  int,
token_count  int,
created_at   timestamptz DEFAULT now()
);

-- email_events
CREATE TABLE IF NOT EXISTS email_events (
event_id             bigserial PRIMARY KEY,
message_id           text,
internet_message_id  text,
from_addr            text,             -- store redacted value only
from_domain          text,
subject              text,
body_plain_redacted  text,
received_at          timestamptz,
bucket               text,
method               text CHECK (method IN ('rule','llm')),
top1_similarity      double precision,
draft_self_score     double precision,
confidence           double precision,
citations            jsonb,
draft_message_id     text,
categories_applied   text[] DEFAULT '{}',
retry_count          int DEFAULT 0,
last_error           text,
status               text CHECK (status IN ('queued', 'processing', 'drafted', 'error')),
created_at           timestamptz DEFAULT now(),
updated_at           timestamptz DEFAULT now()
);

-- Idempotency (handles nulls by coalescing to empty)
CREATE UNIQUE INDEX IF NOT EXISTS email_events_msg_uniq
ON email_events (COALESCE(message_id, ''), COALESCE(internet_message_id, ''));

-- Performance indexes
CREATE INDEX IF NOT EXISTS email_events_received_idx ON email_events (received_at DESC);
CREATE INDEX IF NOT EXISTS email_events_status_idx   ON email_events (status);

-- Vector index (IVFFlat over cosine)
CREATE INDEX IF NOT EXISTS policy_chunks_embed_idx
ON policy_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Optional: for fast section reconstruction
CREATE INDEX IF NOT EXISTS policy_chunks_doc_order_idx
ON policy_chunks (doc_id, chunk_index);

Note: Run ANALYZE after bulk ingestion; IVFFlat benefits from tuning lists to your corpus size. Adjust vector(768) if you change the embedding model.

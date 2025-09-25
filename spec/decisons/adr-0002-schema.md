## adr-0002-schema.md

# ADR-0002: Database Schema (MVP)

## Status
Proposed (2025-09-25)

## Context
Multiple specs reference tables (`email_events`, `policy_docs`, `policy_chunks`), but no single schema document exists. This ADR freezes MVP table names and key fields.

## Decision
We will maintain a simple relational schema in Postgres 16 with pgvector:

### Table: policy_docs
- doc_id (PK, text)
- name (text)
- version (text)
- created_at (timestamptz)

### Table: policy_chunks
- chunk_id (PK, bigserial)
- doc_id (FK → policy_docs.doc_id)
- section (text)
- content (text)
- embedding (vector(768))
- page (int, nullable)

### Table: email_events
- event_id (PK, bigserial)
- message_id (text, unique)
- internet_message_id (text, unique)
- from_addr (text, redacted)
- subject (text)
- body_plain_redacted (text)
- bucket (text)
- method (enum: rule|llm)
- confidence (float)
- citations (jsonb)
- draft_message_id (text, nullable)
- status (enum: queued|processing|drafted|error)
- created_at (timestamptz)
- updated_at (timestamptz)

## Consequences
- Schema provides consistent names across worker, RAG API, and review app.
- Explicit redaction field avoids leaking raw PHI/PII into logs.
- Future migrations can extend (Alembic).

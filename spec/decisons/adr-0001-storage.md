## adr-0001-storage.md

ADR-0001: Storage Choice

Date: 2025-09-25
Status: Accepted
Context

The system must store:
	•	Policy documents and their chunked embeddings for RAG retrieval.
	•	Email events, classifications, draft metadata, and logs.

Requirements:
	•	Support for vector similarity search.
	•	Concurrency (worker + RAG service accessing the same store).
	•	Persistence and durability.
	•	Self-hosted and lightweight enough to run on a Mac Studio (MVP).

Decision

Use PostgreSQL 16 with pgvector as the single shared database for both structured data and embeddings.
	•	Embeddings stored in policy_chunks.embedding column with ivfflat index.
	•	Email metadata and logs stored in email_events.
	•	Both RAG service (FastAPI) and Outlook worker (daemon) connect to the same DB.

Rationale
	•	Already bootstrapped locally for project.
	•	pgvector gives efficient cosine similarity search without a separate vector DB.
	•	Better concurrency and indexing than SQLite.
	•	Simplifies ops — one DB to back up, no extra services.

Alternatives Considered
	•	SQLite + in-memory embeddings: simpler, but poor concurrency and limited vector index support.
	•	Dedicated vector DB (Pinecone, Weaviate, Milvus): more features, but requires extra services/cloud dependency — not aligned with MVP self-hosting.

Consequences
	•	Slightly heavier footprint than SQLite, but acceptable for Mac Studio.
	•	Schema migrations required (handled with Alembic).
	•	Future scalability: can shard/scale Postgres or migrate to managed Postgres if needed.

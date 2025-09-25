Test Plan — Email Drafting (v1)

Purpose
Validate the MVP end-to-end: classification → retrieval → draft generation → Outlook draft write-back → tagging → logging, with safety (no /send), HTML sanitation, and PHI redaction.

Scope
• RAG Service (FastAPI): /health, /search (debug), /draft
• Outlook Worker (daemon): polling, classification (rules → LLM fallback), draft orchestration, tagging, retries
• Postgres + pgvector: documents, chunks, events
• Prompts and contracts under spec/features/email-drafting/v1/*

Out of scope (v1)
• Webhooks/public callbacks, multi-mailbox, auto-attachments, auto-send, non-English

Environments
• Local macOS (developer machine)
• Postgres 16 (+ pgvector)
• Ollama on localhost (models from .env.example)
• Microsoft Graph (delegated auth, test mailbox)

Prereqs
• Postgres running and DB created (rotation_rag)
• pgvector extension enabled
• Ollama running and models pulled (GEN_MODEL, EMBED_MODEL)
• .env configured (see .env.example)
• Microsoft Entra app registered; permissions granted; device-code flow usable
• Schema applied (policy_docs, policy_chunks, email_events, indexes)

Data/fixtures
• Policy corpus: Handbook v2, Policy Manual v1, CME policy PDF
• Email fixtures (EML or JSON): see test-cases.md “Fixtures (suggested)”
• Optional: a seed script to load fixtures via Graph (or send to test mailbox)

Test roles
• Tester: you (owner)
• Systems under test: rag_api, graph_worker, Postgres, Ollama, Graph

High-level flow to run tests
	1.	Reset: stop services, clear or migrate DB (keep embeddings if desired)
	2.	Ingest: run ingestion CLI on 3–5 core policy docs
	3.	Start RAG API: run services/rag_api
	4.	Start Worker: run services/graph_worker (device-code login if needed)
	5.	Seed: deliver fixture emails to inbox (or replay via Graph)
	6.	Observe: verify drafts in Outlook, tags on originals, logs in DB
	7.	Assert: check expectations per test-cases.md
	8.	Record: mark pass/fail and issues

Reset / setup steps
	1.	Database prep
– psql rotation_rag -c “TRUNCATE email_events RESTART IDENTITY;”
– Optional: re-ingest policies if you changed documents
	2.	Ingestion CLI
– make ingest FILE=/path/to/Handbook_v2.pdf
– make ingest FILE=/path/to/Policy_Manual_v1.pdf
– make ingest FILE=/path/to/CME_policy.pdf
Acceptance: policy_docs has 3+ rows; policy_chunks has > 100 rows; ivfflat index exists
	3.	Start RAG API
– make rag (or python -m uvicorn services.rag_api.app:app –reload)
Acceptance: GET /health → { db: “ok”, embed_model: “…”, gen_model: “…” }
	4.	Start Worker
– make worker (or python services/graph_worker/worker.py)
– If first run: complete device-code login in browser
Acceptance: worker logs “polling every 180s”, last checkpoint set

Seeding emails (three options)
A. Manual: send test emails from another account to your test mailbox using the subjects/bodies from test-cases.md
B. Replay: a small helper script that uses Graph “sendMail” to your own inbox (safer to keep this out of MVP; manual is fine)
C. Forward: forward fixture EMLs into the inbox

Execution matrix (reference test-cases.md)
Run TC-01 through TC-16 in order. For each case:

For each test case, capture:
• Input: subject/body (and any setup such as pre-applied category)
• Expected bucket and method (rule|llm)
• Expected citations (Doc:Section)
• Expected confidence threshold behavior
• Safety expectations (no /send; sanitized HTML)
• Logging expectations (status transitions; redactions)

Verification steps (per message)
	1.	Outlook
– Original message categories include “Auto-Drafted” and the expected bucket
– A new Draft reply exists in Drafts for the same thread
– HTML draft contains “Source policy: [Doc:Section; …]”; for low evidence, includes “Next steps”
	2.	Database
– email_events has a row for the message_id/internetMessageId
– Fields populated: bucket, method, confidence, model_id, draft_message_id, rag_citations (JSON), status = drafted
– For redaction tests: stored/redacted fields do not contain full phone or full student email local part
	3.	Logs (service stdout or structured logs)
– No “/send” references anywhere
– For retry tests: backoff intervals logged; terminal status is drafted or error with message

Pass/fail criteria (release gate)
A build is “green” when:
• TC-01..06 pass (core buckets)
• At least one of TC-07 or TC-09 demonstrates required “Next steps” behavior
• TC-10 (HTML safety) passes (no script, no javascript:, no on* attrs)
• TC-11 (PHI/PII redaction) passes
• TC-12 (idempotency) passes
• TC-13 (retry) either passes (eventual success) or fails cleanly with status=error and no draft created
• No test results in any call to /send

Troubleshooting guide
• Worker not picking up emails
– Check checkpoint timestamp; ensure inbox has newer messages
– Verify filter: receivedDateTime ge {last_checkpoint} and isDraft eq false
– Ensure categories filter isn’t excluding fresh messages inadvertently

• Draft missing but DB says drafted
– Check Graph createReply/PATCH responses; ensure HTML not rejected
– Validate that the worker is writing to the correct thread id

• Low confidence everywhere
– Verify embeddings model (EMBED_MODEL) and dimension (768)
– Confirm chunk sizes (800–1200 chars) and overlap (150)
– Inspect top_k passages via /search to ensure retrieval is sane

• HTML contains suspicious content
– Confirm sanitizer enforced: strip , javascript: URLs, on* attributes
– Make sure user body is escaped before inclusion

• Repeated processing of same email
– Ensure unique constraint on (message_id, internet_message_id)
– Skip items already tagged Auto-Drafted

• Graph 401/invalid_grant
– Re-run device-code login; verify MS_CLIENT_ID/MS_TENANT_ID; check token persistence location

Evidence to store
• Screenshot of Outlook categories and the Draft for at least TC-01, TC-05, TC-10
• Copy of RAG API /health output for the test session
• email_events rows (selected fields) for TC-07, TC-11, TC-13
• Worker logs snippet showing a retry/backoff sequence (TC-13)

Exit report template
• Run date/time, environment, commit hash
• Summary table (TC-01..TC-16 with pass/fail and notes)
• Open defects and severity
• Recommendation: proceed / block release

Maintenance of the plan
• Update when: model choice changes, schema changes, or additional buckets are added
• Re-baseline timings if hardware changes (e.g., moving to Mac Studio)

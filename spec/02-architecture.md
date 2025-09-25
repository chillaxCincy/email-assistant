# Architecture (MVP)

## 1) Processes & Communication
- **Services (2 processes):**
  - **RAG Service** — FastAPI over HTTP (REST).
  - **Outlook Worker** — daemon/cron-like process with no public API in MVP.
- **Shared state:** PostgreSQL 16 + `pgvector` (no queues in MVP).
- **Rationale:** Simplest ops; no need for Worker HTTP endpoints.

## 2) Vector Store & Embeddings
- **DB:** Postgres 16 with `pgvector`.
- **Embeddings:** `nomic-embed-text` via Ollama (dim **768**).
- **Similarity:** cosine; **top_k = 6** (configurable).

## 3) Database Choice
- **Postgres** (not SQLite) for concurrency and IVFFlat index support.

## 4) Error Handling, Retries, Idempotency
- **Graph backoff:** 1m → 4m → 15m → 60m, max 5 tries, then `status='error'`.
- **LLM/RAG timeouts:** 60s; 2 retries (15s then 45s).
- **Idempotency:** de-dupe on `message_id` / `internetMessageId`.

## 5) Classification (Rules first; LLM fallback)
- **Buckets:** `Rotation Policy`, `CME & Attendance`, `Honorarium & W-9`, `Evaluations/Grades`, `Grand Rounds Logistics`, `Scheduling`, `Misc`.
- **Keyword patterns (case-insensitive):**
  - **Honorarium & W-9:** `w-9|w9|honorarium|speaker fee|engagement letter|tax form`
  - **CME & Attendance:** `cme|credit|attendance (form)?|certificate|qr code|sign[- ]in`
  - **Evaluations/Grades:** `evaluation|eval|assessment|grade|rubric|score|feedback`
  - **Grand Rounds Logistics:** `grand rounds|flyer|speaker|talk|abstract|poster`
  - **Scheduling:** `schedule|reschedul|availability|make[- ]up|swap|calendar`
  - **Rotation Policy:** `policy|procedure|handbook|syllabus|guideline|requirements`
- **Fallback:** if multiple/no matches, call LLM to select **exactly one** label.

## 6) Confidence (single scalar for review)
- **Retrieval:** `top1_similarity` (0–1).
- **Classification:** 1.0 for rule; else model self-score (0–1).
- **Draft:** model self-score clipped by retrieval quality.
- **Overall (reported):** `min(top1_similarity, draft_self_score)`.
- **Low-confidence threshold:** **0.55** → flag for re-draft/manual review.

## 7) Citations
- **Draft format:** `Source policy: [Doc:Section; Doc:Section]`.
- **Display:** no chunk IDs in emails; page numbers allowed in section text.
- **Logs:** store `doc_id`, `section`, optional `page`, and `chunk_id`.

## 8) PHI Redaction
- Treat as PHI/PII in logs: student full names, phone numbers, student emails, MRNs.
- **Logging policy:** redact phone numbers + student email local-part; keep sender domain and initials. Don’t store attachments by default.

## 9) Outlook Worker Details
- **Polling cadence:** default **180s** (env-configurable).
- **Time basis:** store/compare timestamps in **UTC**.
- **Filter:** `receivedDateTime ge {last_checkpoint} and isDraft eq false`.
- **Pagination:** follow `$skiptoken` until empty; process newest-first.
- **Write-back:** `createReply` → `PATCH` body with HTML; add categories `Auto-Drafted` + bucket; **never** call `/send`.

## 10) RAG Service Endpoints (contracts)
- **POST `/draft`**
  - **Request:**
    ```json
    { "subject": "string", "body": "string", "top_k": 6 }
    ```
  - **Response:**
    ```json
    {
      "html": "<p>…</p><p><strong>Source policy:</strong> [Handbook:CME Requirements; Policy Manual:Attendance]</p>",
      "citations": [
        { "doc_id": "handbook_v2", "section": "CME Requirements", "score": 0.83 },
        { "doc_id": "policy_manual_v1", "section": "Attendance", "score": 0.71 }
      ],
      "confidence": 0.72,
      "model_id": "ollama/llama3.1:8b-instruct-q4_K_M",
      "query_time_ms": 240
    }
    ```
  - **Rules:** always include the **Source policy** line; add **Next steps** when evidence is weak.

- **POST `/search`** *(debug only)*
  - **Request:** `{ "query": "string", "top_k": 6 }`
  - **Response:**
    ```json
    {
      "passages": [
        {
          "chunk_id": 12345,
          "doc_id": "handbook_v2",
          "section": "CME Requirements",
          "content": "…",
          "score": 0.87
        }
      ]
    }
    ```

- **GET `/health`**
  - Returns DB connectivity + model names + service version, e.g.:
    ```json
    { "db":"ok", "embed_model":"nomic-embed-text", "gen_model":"llama3.1:8b-instruct-q4_K_M", "version":"0.1.0" }
    ```

## 11) Security & HTML Safety
- **Secrets:** `.env` only; never commit secrets.
- **Scopes:** `Mail.Read`, `Mail.ReadWrite`, `offline_access` (delegated).
- **HTML:** sanitize generated draft (strip scripts/unsafe attributes); ensure UTF-8 and a lightweight inline CSS wrapper.

## 12) DB Indexes (performance)
- `CREATE INDEX email_events_received_idx ON email_events (received_at DESC);`
- `CREATE INDEX email_events_status_idx   ON email_events (status);`
- `CREATE INDEX policy_chunks_embed_idx   ON policy_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists=100);`

## 13) Out of MVP (explicit)
- Webhooks/public callbacks, auto-attach forms, multi-mailbox, auto-send, non-English handling, queues/streaming.

## 14) References to Other Specs
- **Env template:** see `.env.example`.
- **Prompts:** see `spec/contracts/prompt.md`, `spec/contracts/classifier.md`.

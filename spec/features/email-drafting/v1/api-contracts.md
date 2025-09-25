## api-contracts.md

# API Contracts for Email Drafting (v1)

## RAG Service

### POST /draft
Request:
{
  "subject": "string",
  "body": "string",
  "top_k": 6
}

Response:
{
  "html": "<p>...</p><p><strong>Source policy:</strong> [Handbook:CME Requirements; Policy Manual:Attendance]</p>",
  "citations": [
    { "doc_id": "handbook_v2", "section": "CME Requirements", "score": 0.83 },
    { "doc_id": "policy_manual_v1", "section": "Attendance", "score": 0.71 }
  ],
  "confidence": 0.72,
  "model_id": "ollama/llama3.1:8b-instruct-q4_K_M",
  "query_time_ms": 240
}

Notes:
- Always include the "Source policy" line.
- Add "Next steps" if evidence is weak (overall confidence < 0.55).
- confidence = min(top1_similarity, draft_self_score)
  • top1_similarity: cosine similarity of the highest-scoring retrieved passage
  • draft_self_score: model’s self-reported confidence in its draft (0–1)

---

### POST /search (debug only)
Request:
{
  "query": "string",
  "top_k": 6
}

Response:
{
  "passages": [
    {
      "chunk_id": 12345,
      "doc_id": "handbook_v2",
      "section": "CME Requirements",
      "content": "CME credits must be submitted within 30 days...",
      "score": 0.87
    }
  ]
}

---

### GET /health
Response:
{
  "db": "ok",
  "embed_model": "nomic-embed-text",
  "gen_model": "llama3.1:8b-instruct-q4_K_M",
  "version": "0.1.0"
}

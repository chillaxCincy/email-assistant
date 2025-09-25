	•	Components: RAG API, Outlook worker, Vector DB (pgvector), Doc ingestor, Reviewer UI.
	•	Interfaces:
	•	RAG API: POST /draft { subject, body } → { html, citations[], confidence }
	•	Outlook write-back: createReply then PATCH body (no /send).
	•	Model choices (local MVP): llama3.1:8b-instruct-q4_K_M for drafting; nomic-embed-text for embeddings.

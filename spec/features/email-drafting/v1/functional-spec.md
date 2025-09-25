## functional-spec.md

Functional Specification — Email Drafting (v1)

Scope

Classify new Outlook emails, retrieve relevant policy excerpts, generate an HTML draft with citations, save as a Draft reply, and tag the original message. No auto-send.

Inputs
	•	Outlook message: id, internetMessageId, from, subject, body (plain text preferred), receivedDateTime
	•	Policy index: chunked sections with doc_id, section, embedding

Outputs
	•	Draft HTML saved to Outlook Drafts for the same thread
	•	Categories on original: “Auto-Drafted” and one of the buckets
	•	Log row in email_events with citations and confidence

Processing Flow
	1.	Intake
	•	Poll every 180s; filter new, non-draft messages (UTC checkpoint)
	•	De-duplicate by message_id or internetMessageId
	2.	Classification
	•	Apply regex rules; if ambiguous, run LLM fallback to choose exactly one bucket
	3.	Retrieval
	•	Query vector store with subject + body; top_k=6 passages; record scores
	4.	Drafting
	•	Prompt model with email + passages
	•	Must include “Source policy: [Doc:Section; …]”; add “Next steps” when evidence is weak
	•	Compute confidence = min(top1_similarity, draft_self_score)
	5.	Write-back
	•	createReply → PATCH HTML body
	•	PATCH categories on original
	6.	Logging
	•	Store status transitions (queued → processing → drafted|error), citations (doc_id, section, score), timing, confidence, model_id
	7.	Idempotency
	•	Skip if message already processed or already tagged Auto-Drafted

Buckets

Rotation Policy; CME & Attendance; Honorarium & W-9; Evaluations/Grades; Grand Rounds Logistics; Scheduling; Misc

Business Rules
	•	Never invoke /send
	•	Require at least one citation; aim for ≥2 when available
	•	If top1_similarity < 0.55, include a Next steps line
	•	Redact PHI/PII in logs (student emails local-part, phone numbers)

Configuration
	•	Poll interval: default 180s
	•	top_k: default 6
	•	low-confidence threshold: 0.55
	•	Models: GEN_MODEL, EMBED_MODEL (from env)

Failure Handling
	•	Graph/API errors: exponential backoff 1m → 4m → 15m → 60m, max 5
	•	LLM timeouts: 60s; retry 2 times (15s, 45s)
	•	On final failure: status=error with error message; do not create draft

Observability (MVP)
	•	Metrics captured in logs: processed count, errors, avg latency, % low-confidence
	•	Simple reviewer page can query email_events table (added later)

⸻

If you want, I can follow with concise drafts for:
	•	nonfunctional-spec.md (performance, security, reliability, maintainability)
	•	test-cases.md (the 6–8 canonical emails + assertions)
	•	03-test-plan.md (how/when to run, fixtures, acceptance gates)
	•	04-acceptance-criteria.md (done/not-done checklist)
	•	backlog.md (post-MVP items like attachments helper, nightly polish, Teams digest)

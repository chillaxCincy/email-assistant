## 01-requirements.md
Requirements — Email Assistant (MVP)

1) Goal

Automatically classify incoming Outlook emails related to the neurology rotation, perform RAG over internal policy/procedure documents, and save Draft replies with citations for human review — never auto-send.

2) Scope (v1)
	•	Acquisition: Poll Outlook inbox every 3–5 minutes (no webhooks).
	•	Classification: Bucket each email into 5–7 categories:
	•	Rotation Policy, CME & Attendance, Honorarium & W-9, Evaluations/Grades, Grand Rounds Logistics, Scheduling, Misc.
	•	Drafting (RAG): Retrieve top-k policy chunks and produce an HTML reply including:
	•	concise answer,
	•	“Next steps” if info is missing,
	•	Source policy line with [Doc:Section] citations (≥2 when possible).
	•	Tagging: Apply Outlook Categories to the original message (e.g., Auto-Drafted, plus the bucket).
	•	Persistence: Log runs to a local DB (inputs, outputs, citations, confidence, errors).
	•	Review: Minimal reviewer page to list processed emails, show citations, and link to the Outlook Drafts item.

3) Out of Scope (v1)
	•	Webhooks / public callbacks.
	•	Automatic attachments (e.g., auto-attaching W-9/CME forms).
	•	Multiple mailboxes / shared mailboxes.
	•	Auto-sending or scheduling sends.
	•	Non-English handling.

4) Stakeholders
	•	Owner/Operator: James Wise (single user for MVP).
	•	Consumers: Students/colleagues via replies you send after review.

5) Constraints
	•	Self-hosted/local first (Mac laptop/desktop).
	•	Zero auto-send policy; system must be physically unable to send in MVP.
	•	PHI minimization: do not extract or store PHI beyond what arrives in the email; redact in logs where feasible.

6) Functional Requirements

6.1 Email Intake
	•	Poll Microsoft Graph for new, non-draft messages since last checkpoint.
	•	De-duplicate by message_id / internetMessageId.
	•	Persist: from, subject, received time, plain-text body (sanitized), and processing status.

6.2 Classification
	•	Apply rule-first classification; if ambiguous, use a small LLM fallback to return exactly one label from the category list.
	•	Store chosen bucket; include rule vs LLM provenance.

6.3 Retrieval-Augmented Drafting
	•	Query vector store with email subject+body; fetch top-k = 6 passages.
	•	Prompt the local model to produce HTML with:
	•	greeting + one-sentence summary,
	•	clear answer (bullets permitted),
	•	“Source policy: [Doc:Section; …]”,
	•	“Next steps:” when policy is insufficient/ambiguous.
	•	Return citations list with document identifiers and similarity scores.

6.4 Outlook Write-Back
	•	Create Draft via createReply; update body via PATCH.
	•	Tag original message with categories: Auto-Drafted + bucket.
	•	Never call /send.

6.5 Review UX (minimal)
	•	Table of processed emails: subject, from, bucket, confidence, status (queued/drafted/error).
	•	Detail view: original text, retrieved passages, rendered HTML draft, citation list, and link Open in Outlook Drafts.

7) Non-Functional Requirements
	•	Reliability/SLO: ≥99% of new emails processed within 10 minutes.
	•	Safety: No code path invokes /send; CI (later) should flag any /send references.
	•	Security: Secrets via env; least-privilege Graph scopes (Mail.Read, Mail.ReadWrite, offline_access if delegated).
	•	Cost: $0 infra (local), no external AI APIs required.
	•	Maintainability: .env config; components decoupled by simple HTTP contracts.

8) Success Criteria (MVP Acceptance)
	•	Quality: ≥90% of rotation-policy emails produce a usable draft with ≥2 citations.
	•	Safety: 0 auto-send events.
	•	Traceability: Every draft includes a Source policy line; logs contain citation IDs.
	•	Operability: Start/stop scripts run locally; daily review takes ≤10 minutes.

9) Data Requirements
	•	Vector store content: policy/procedure docs only (PDF/DOCX/MD → chunked text + embeddings). No student PHI in the index.
	•	Email logs: store minimal necessary fields; redact signatures/phones in logs when feasible.
	•	Backups: daily DB dump (local).

10) Assumptions
	•	Single user mailbox with rights to create Drafts and apply Categories.
	•	English-only policy documents and emails.
	•	Local models via Ollama (7–13B for day work; optional 70B nightly polish later).

11) Risks & Mitigations
	•	Policy drift: docs change without re-embedding → schedule a weekly re-ingest.
	•	Hallucination risk: enforce “answer only from provided excerpts”; require citations; add “Next steps” when unsure.
	•	Throughput on small hardware: keep concurrency = 1; polling cadence 3–5 min.
	•	Graph auth expiry: use device-code or client-credentials; refresh tokens securely.

12) Metrics (MVP)
	•	Emails processed/hour; time from arrival → draft.
	•	% low-confidence drafts (threshold default 0.55).
	•	Average # of citations per draft.
	•	Error rate (Graph/LLM/vector ops).

13) Test Scenarios (must pass)
	1.	CME request → bucket CME & Attendance, draft with 2+ citations.
	2.	W-9 / honorarium → correct bucket, draft includes required form mention.
	3.	Evaluations/Grades → references correct policy section.
	4.	Grand Rounds logistics → includes date/time/location guidance when available.
	5.	Scheduling question → points to policy (e.g., make-up rules), asks for missing details.
	6.	Ambiguous email → returns “Next steps” request and cites at least 1 source.
	7.	Safety → verify there is no /send anywhere; original message gets categories; draft saved.

14) Out-of-Band SOPs
	•	Daily: review low-confidence and error items; send from Outlook.
	•	Weekly: re-ingest changed docs; review metrics; vacuum/analyze DB.

⸻

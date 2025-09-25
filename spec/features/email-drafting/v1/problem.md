## problem.md

Problem — Email Drafting for Rotation Policies (v1)

Context

You manage a high volume of Outlook emails from students and faculty about a neurology rotation. Most questions are repetitive and answerable from a small set of policy/procedure documents (handbook, attendance/CME, evaluations, scheduling rules, honorarium/W-9, grand rounds logistics).

Problem Statement

Incoming emails are manually triaged and answered one-by-one. This is slow, error-prone (misremembered policy, missed updates), and interrupts focused work. You need an assistant that classifies messages, pulls the right policy text, and prepares draft replies you can quickly review and send—without ever auto-sending.

Primary Users & Jobs-To-Be-Done
	•	Clerkship Director / Coordinator (you)
	•	JTBD 1: Quickly find the correct policy passage relevant to an email.
	•	JTBD 2: Produce a concise, correct reply that cites source policy.
	•	JTBD 3: Keep a record of how each email was handled (traceability).
	•	Secondary: Students/faculty (indirect beneficiaries of faster, more consistent answers).

Pain Points Today
	•	Repetitive Q&A across many threads.
	•	Time lost searching documents; answers drift from actual policy.
	•	No uniform citation to source; hard to audit decisions.
	•	Manual tagging/organization in Outlook; drafts start from scratch.
	•	Context switches; delayed replies.

Goals (What success looks like)
	•	Reduce time-to-draft per email from minutes to seconds.
	•	Ensure every answer references the correct policy section.
	•	Standardize tone and structure while allowing final edits.
	•	Keep operation fully self-hosted; never auto-send.

Non-Goals (v1)
	•	Webhooks/callback exposure.
	•	Auto-attaching forms or files.
	•	Multi-mailbox or shared mailboxes.
	•	Auto-sending or scheduling sends.
	•	Non-English support.

Proposed Solution (High Level)
	•	Always-on worker polls Outlook, classifies each new message (rules → LLM fallback).
	•	RAG service retrieves top policy excerpts and generates an HTML draft with a mandatory “Source policy: [Doc:Section; …]” line and a “Next steps” section when evidence is weak.
	•	Worker saves a Draft reply (never sends) and applies Outlook Categories for tracking. All runs are logged with citations and confidence.

Constraints / Assumptions
	•	Self-hosted on macOS; no third-party LLM APIs required.
	•	Local models via Ollama (8–13B for day work; optional larger model offline).
	•	Documents are English and reasonably structured.
	•	PHI minimization in logs; no attachment storage by default.

Risks & Mitigations
	•	Policy drift: schedule weekly re-ingest/re-embed of docs.
	•	Hallucination: prompt constrains to excerpts; citation line required; low-confidence flagged.
	•	Misclassification: rule-first approach with LLM fallback; allow manual bucket change in review.
	•	Throughput on small hardware: single worker, 3–5 min polling; long jobs backoff/retry.

Success Metrics (MVP)
	•	≥90% of rotation-policy emails produce a usable draft with ≥2 citations.
	•	0 auto-send events.
	•	Median time from arrival → draft < 5 minutes (polling-based).
	•	≤5% drafts require major rewrite.

Example User Stories
	1.	As a clerkship director, when a student asks about CME credit, I want a draft that cites the CME section and includes next steps if details are missing.
	2.	As a clerkship director, when someone asks to reschedule, I want a draft citing make-up rules and requesting specific dates to proceed.
	3.	As a clerkship director, when asked about honorarium/W-9, I want a draft pointing to the correct form and submission steps with citations.
	4.	As a clerkship director, I want all processed emails tagged and logged so I can audit what policy was used.

Current Workarounds (Baseline)
	•	Manual search in PDFs/handbook; ad-hoc replies; optional Outlook categories. No standardized citations; variable tone and completeness.

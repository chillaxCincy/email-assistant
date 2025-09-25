Test Cases — Email Drafting (v1)

Note: All times UTC. All messages are new, non-draft, from the primary inbox. Unless stated, policy corpus contains Handbook v2, Policy Manual v1, CME PDF v1.

========================================
TC-01 CME credit request (happy path)

Intent: Classify as “CME & Attendance”, retrieve CME policy, produce draft with ≥2 citations.

Input (subject/body):
Subject: CME credit for last Friday’s session
Body: Hi — how do I get CME credit for last Friday? Is there a form or deadline?

Preconditions:
– CME policy contains “Attendance must be submitted within 30 days” and “Use CME Form A”.

Expected:
– Bucket = CME & Attendance (method=rule, confidence=1.0)
– Retrieval top1_similarity ≥ 0.45
– Draft HTML includes: deadline + form name
– “Source policy: [Handbook:CME Requirements; Policy Manual:Attendance]”
– Categories applied: Auto-Drafted, CME & Attendance
– Overall confidence ≥ 0.60
Assertions:
	1.	Classification == CME & Attendance
	2.	Draft contains “30 days” and “Form A”
	3.	Citations include CME Requirements
	4.	Status == drafted and draft_id present

========================================
TC-02 W-9 / Honorarium (happy path)

Intent: “Honorarium & W-9” via rules.

Input:
Subject: W-9 for recent lecture
Body: Hi, where should I submit my W9 for the honorarium?

Expected:
– Bucket = Honorarium & W-9 (rule, 1.0)
– Draft references “W-9 submission steps” and “finance@…”
– Source policy line with at least one W-9 section citation
– Categories: Auto-Drafted, Honorarium & W-9
– Overall confidence ≥ 0.60

========================================
TC-03 Evaluations / Grades (happy path)

Input:
Subject: Evaluation timeline
Body: When will my rotation evaluation be available, and where do I see the grade?

Expected:
– Bucket = Evaluations/Grades
– Draft explains evaluation release window and viewing portal
– ≥2 citations including Evaluations section
– Confidence ≥ 0.60

========================================
TC-04 Grand Rounds logistics (happy path)

Input:
Subject: Grand Rounds speaker flyer
Body: Can you share the speaker flyer and time for next week’s grand rounds?

Expected:
– Bucket = Grand Rounds Logistics
– Draft states location/time policy or points to calendar link as per policy
– Citations include Grand Rounds section
– Confidence ≥ 0.55 (may be lower if policy brief)

========================================
TC-05 Scheduling (make-up rules)

Input:
Subject: Missed clinic yesterday
Body: I missed my clinic yesterday; how do I schedule a make-up?

Expected:
– Bucket = Scheduling
– Draft cites make-up rules and requests specific dates (Next steps)
– Citations include Scheduling/Make-up section
– Confidence ≥ 0.55

========================================
TC-06 General rotation policy question

Input:
Subject: Rotation attendance requirement
Body: How many days can I miss during the rotation?

Expected:
– Bucket = Rotation Policy
– Draft states attendance allowance and escalation path
– ≥2 citations from Handbook and Policy Manual
– Confidence ≥ 0.60

========================================
TC-07 Ambiguous inquiry → Next steps required

Input:
Subject: Question about requirements
Body: I need to know what I have to do for this month.

Expected:
– Likely Bucket = Rotation Policy (LLM fallback)
– Retrieval top1_similarity < 0.35 triggers explicit “Next steps” asking for specifics
– At least 1 citation if any relevant snippet; otherwise still include “Source policy” with whatever closest section is found
– Overall confidence < 0.55 → flagged low-confidence

Assertions:
	1.	method == llm
	2.	Draft contains a “Next steps” request
	3.	Confidence < 0.55 and item flagged

========================================
TC-08 Mixed topics (conflict resolution)

Input:
Subject: CME and make-up clinic
Body: I need CME credit and also to reschedule a missed clinic.

Expected:
– If rules tie, LLM selects one primary bucket; acceptable outputs: CME & Attendance OR Scheduling
– Draft addresses both topics with separate bullets
– Citations include at least one from each relevant section
– Confidence ≥ 0.55

========================================
TC-09 Low-similarity retrieval but clear classification

Input:
Subject: CME
Body: CME???

Expected:
– Bucket = CME & Attendance (rule)
– Retrieval may be weak (short body); draft must still include “Next steps”
– At least 1 citation if possible; confidence may be ~0.45–0.55
– Categories applied; status drafted

========================================
TC-10 HTML safety (injection attempt)

Input:
Subject: alert(1) CME
Body: Please click here.

Expected:
– Bucket = CME & Attendance
– Generated HTML sanitizes subject/body: no script tags, no javascript: URLs, no on* attributes
– Draft still valid and cites policy
Assertions:
	1.	Draft contains sanitized subject text (escaped)
	2.	No “<script” nor “javascript:” nor onload= in output

========================================
TC-11 PHI/PII redaction in logs

Input:
Subject: Absence for appointment
Body: Hi, I (john.smith@student.university.edu, (555) 123-4567) missed clinic.

Expected:
– Bucket = Scheduling
– In email_events logs: student email local part redacted (e.g., j***@student.university.edu); phone redacted or masked
– Draft itself may echo contact info only if present in email; logs must be redacted
Assertions:
	1.	Stored body_plain_redacted does not contain full phone or full student email
	2.	Categories and draft saved as usual

========================================
TC-12 Idempotency (duplicate processing)

Setup:
– Insert a message with message_id X, already processed (status=drafted)

Input:
– Polling run sees the same message X again

Expected:
– Worker skips processing due to uniq constraint / seen checkpoint
– No new draft created; no duplicate log rows

========================================
TC-13 Graph API failure with retries

Setup:
– Simulate transient 429 or 5xx on createReply

Expected:
– Worker retries with backoff 1m → 4m → 15m; succeeds within retry budget
– On eventual success, status=drafted
– If all retries exhausted, status=error with last exception message; no draft id

========================================
TC-14 Large email body (performance)

Input:
Subject: Scheduling details
Body: 10–20 KB of text (pasted thread)

Expected:
– Bucket = Scheduling by rules or fallback
– System truncates/normalizes input for embedding as needed
– Draft generated within 60s; confidence computed; citations present

========================================
TC-15 Already tagged message (skip)

Setup:
– Original message already has category “Auto-Drafted”

Input:
– Polling run includes that message

Expected:
– Worker skips processing to avoid loop
– No changes written

========================================
TC-16 Attachment mention (no auto-attach in MVP)

Input:
Subject: W-9 form
Body: Please attach the W-9 to your reply.

Expected:
– Bucket = Honorarium & W-9
– Draft may include a line “Attachments to include: W-9 form” but does not attach files
– Citations present; categories applied

========================================
Fixtures (suggested)

Provide 8–10 sample emails as .eml or JSON (subject, body, from, receivedDateTime) covering all buckets and edge cases (TC-07, TC-10, TC-11, TC-13). Include at least:
– cme_request.eml
– w9_request.eml
– eval_timeline.eml
– grand_rounds_flyer.eml
– missed_clinic_makeup.eml
– generic_policy_question.eml
– ambiguous_short.eml
– html_injection.eml
– pii_redaction.eml
– graph_retry_fixture.json (to simulate 429/5xx)

========================================
Pass/Fail Criteria (MVP)

A release passes when:
– TC-01..06 all pass
– At least one of TC-07 or TC-09 demonstrates “Next steps” behavior
– TC-10 (HTML safety) passes
– TC-11 (redaction) passes
– TC-12 (idempotency) passes
– TC-13 (retry) either passes (success after retry) or fails cleanly with status=error and no draft sent
– No test produces a call to /send

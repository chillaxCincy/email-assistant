	•	Buckets: Rotation Policy, CME & Attendance, Honorarium & W-9, Evaluations/Grades, Grand Rounds Logistics, Scheduling, Misc.
	•	RAG params: top-k=6; chunk 800–1200 chars; overlap 150; cosine similarity.
	•	Prompt rules: “Answer only from provided excerpts; add ‘Source policy: [Doc:Section; …]’; add ‘Next steps’ if info missing; never send.”
	•	Retry/idempotency: Exponential backoff on Graph; message_id uniqueness in DB.

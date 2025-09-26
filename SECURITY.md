# SECURITY.md

This project is an MVP system for drafting (not sending) replies to student and faculty emails.

## Core Principles
- **Drafts only**: The system never calls `/send`. All outputs are saved as Outlook drafts for manual review.
- **PHI/PII minimization**:
  - Email addresses are redacted before storage (local part masked).
  - Phone numbers and full names are masked in logs.
  - Only non-sensitive policy content is embedded for retrieval.
- **Sanitized outputs**: Draft HTML is escaped to block script injection.
- **Retry limits**: API retries are capped to prevent runaway loops.

## Development Notes
- Secrets must be stored in `.env` and not committed.
- Contributors should run `make no-send-check` in CI to verify no accidental `/send` calls.
- See `docs/security.md` for extended security guidance.

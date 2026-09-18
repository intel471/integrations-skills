# Page atomicity & retry

The rule that keeps a scheduled connector from losing or duplicating events across runs:

> **Advance the persisted cursor only after an entire page has been shipped successfully.**

## Pattern
- Drain pages until one comes back short — fewer items than the requested `size` (see
  `intel471-api-patterns` → [`verity-pagination`](../../intel471-api-patterns/references/verity-pagination.md)
  / [`titan-pagination`](../../intel471-api-patterns/references/titan-pagination.md) for the
  "done" signal per backend).
- For each page: ship **all** events, then persist `cursor = resp.cursor_next`.
- **Mid-page failure ⇒ do not advance.** Return the last fully-shipped cursor in state and let
  the run error out. The next scheduled tick replays from that cursor cleanly — at worst the
  last page is re-shipped (at-least-once), never skipped.

## Retry / backoff (destination side)
- Retry transient destination failures (e.g. webhook 5xx / timeouts) with exponential backoff —
  1s / 2s / 4s is a reasonable default for a connector on a 15-minute timer.
- **Fast-fail on 4xx** (bad request / auth) — retrying won't help; surface the error so the
  scheduler or workflow flags it.

## Consequence to design for
This is **at-least-once** delivery. Downstream should tolerate duplicate events (dedupe on a
stable id, or accept idempotent writes) because a mid-page failure re-ships the last page.

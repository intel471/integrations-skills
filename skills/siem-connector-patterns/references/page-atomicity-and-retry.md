# Page atomicity & retry

The rule that keeps a scheduled connector from **losing** events across runs. It does not
prevent duplicates — those are the accepted trade-off, see *Consequence to design for* below:

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

## Retry / backoff
- **Retry transient failures** — 5xx and timeouts, on the Intel 471 side and the destination
  side alike. Exponential backoff; 1s / 2s / 4s is a reasonable default for a connector on a
  15-minute timer.
- **Retry 429 as well, honouring `Retry-After` when present.** 429 is a 4xx, but it means "try
  again later", so it belongs on the retry path and not the fast-fail path. Neither SDK retries
  it for you — see `intel471-api-patterns` →
  [`rate-limits-retries-errors`](../../intel471-api-patterns/references/rate-limits-retries-errors.md).
- **Fast-fail on the remaining 4xx** — 400 (bad request), 401/403 (auth), 404. Retrying won't
  help; surface the error so the scheduler or workflow flags it.

## Consequence to design for
This is **at-least-once** delivery. Downstream should tolerate duplicate events (dedupe on a
stable id, or accept idempotent writes) because a mid-page failure re-ships the last page.

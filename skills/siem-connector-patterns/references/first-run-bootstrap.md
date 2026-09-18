# First-run bootstrap

On the very first run there's no persisted cursor, so the connector needs a **start time**.

## Conventions worth adopting
- **Default to "now"** when no start time is given, so a fresh install doesn't backfill the
  entire history by accident. This is a *connector* convention, not an API default: omitting
  `from` on the request itself streams **every object from the beginning**, which is what you
  want for a deliberate full pull and not what you want on an accidental one.
- **Accept an explicit start** for controlled backfill. Names used by existing Intel 471
  connectors, if you want to stay recognisable to users who run more than one:
  - `created_after` (Splunk) — a 13-digit **epoch-milliseconds** value is detected with the
    regex `^\d{13}$`, otherwise the value is parsed as a date string.
  - `LookBackDays` (Sentinel Logic Apps).
  - `added_after` / `bootstrap_added_after` (Rapid7) — first-run start, defaulting to **UTC now
    minus a 60-second safety margin**.
  - An initial date **capped** to a bounded window (ServiceNow caps at ≈90 days) to stop the
    first pull running away.
- **Freeze the bootstrap start into state** alongside the cursor and filters, and never send a
  different value on a later run — `from` is frozen for the life of the stream, only the cursor
  advances (`intel471-api-patterns` →
  [`verity-pagination`](../../intel471-api-patterns/references/verity-pagination.md)). To change
  it later, delete/reset the persisted state so the connector re-bootstraps.

## Traps
- **Seconds vs milliseconds.** Verity wants epoch **milliseconds** (13 digits). A seconds value
  bootstraps at 1970 and floods the pipeline. See `intel471-api-patterns` →
  [`date-time-handling`](../../intel471-api-patterns/references/date-time-handling.md).
- **The 60-second margin matters.** Bootstrapping at exactly "now" can miss records whose
  server-side timestamp is a hair behind the client clock. Back off a minute.

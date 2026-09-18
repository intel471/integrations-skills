---
name: intel471-api-patterns
description: Patterns and traps for building integrations against the Intel 471 APIs — the Titan and Verity471 backends and their SDKs. Covers pagination, cursors, auth, rate limits, filtering, and known traps. Consult this whenever writing or debugging code that calls the Titan or Verity471 API/SDK, especially anything that pages through results (cursor / cursor_next / count+offset).
---

# Intel 471 API Integration Patterns

Field-tested knowledge for calling the Intel 471 APIs. **This file is a lean index — read the linked reference file for the specific topic you're working on.** Details stay in `references/` so this index is cheap to keep loaded.

## Backends at a glance

- **Titan** — the primary/original API. Three different paging styles coexist (cursor/stream, `count`+`offset`, and a hybrid count-offset-plus-timestamp workaround).
- **Verity471** — a newer, stream-first subset of Titan. **Cursor-only** pagination.

⚠️ Titan and Verity471 are **not** interchangeable. Field names and paging semantics differ (e.g. Titan streams expose `cursorNext`; the Verity471 SDK exposes `cursor_next`). Confirm which backend you're on before copying code between projects.

## Topics

### Pagination
- **Verity471 cursor pagination** → [`references/verity-pagination.md`](references/verity-pagination.md)
- **Titan pagination — 3 styles + the `/alerts` uid-offset trap** → [`references/titan-pagination.md`](references/titan-pagination.md)

### Auth
- **Auth mechanism, credentials, hosts** → [`references/credentials-and-auth.md`](references/credentials-and-auth.md)

### Filtering & data types
- **Date/time filter formats (epoch-ms, the ISO-date trap)** → [`references/date-time-handling.md`](references/date-time-handling.md)

### Porting between backends
- **Titan ↔ Verity471 differences that break copy-pasted code** → [`references/titan-vs-verity-porting.md`](references/titan-vs-verity-porting.md)

### Rate limits, retries, errors
- **Retries, typed exceptions, CORS** → [`references/rate-limits-retries-errors.md`](references/rate-limits-retries-errors.md)

### STIX
- Intel 471 → STIX 2.1 mapping is **not** documented here. The source of truth is the SDK's own mappers: <https://github.com/intel471/verity471-python/tree/main/verity471/verity_stix>.

## Official documentation

These skills capture the traps, not the full API surface. For endpoint and field
reference:

- **Openly readable** — the Python SDKs, which are the practical source of truth for
  method names, parameter types and response models:
  <https://github.com/intel471/titan-client-python> ([PyPI](https://pypi.org/project/titan-client/)),
  <https://github.com/intel471/verity471-python> ([PyPI](https://pypi.org/project/verity471/)).
- **Customer login required** — the Intel 471 Developer Portal
  (<https://developer.intel471.com/>), which also issues API credentials, and the Titan
  API reference at <https://titan.intel471.com/api/docs/>.

## How to extend this skill

1. Create `references/<topic>.md` with the detail.
2. Add a one-line link to it under the right heading above.
3. Open a pull request. Keep this `SKILL.md` short — only this index needs to stay loaded; the reference files load on demand.

See [`CONTRIBUTING.md`](../../CONTRIBUTING.md) for the full flow.

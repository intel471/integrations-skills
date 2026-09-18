---
name: siem-connector-patterns
description: Reusable architecture for scheduled Intel 471 → SIEM/platform connectors (Splunk, Rapid7 InsightConnect, Microsoft Sentinel, ServiceNow, and your own) — persisting a cursor + frozen filters between runs, incremental drain-until-short-page pulls, first-run bootstrap, page-level atomicity, and the Titan non-stream timestamp-replay trap. Consult when building or debugging any connector that pulls a Titan/Verity471 stream on a timer.
---

# Intel 471 → SIEM Connector Patterns

Almost every Intel 471 SIEM/platform connector has the same shape: a **scheduled job that
incrementally pulls a Titan/Verity471 stream and pushes into a destination, persisting where it
left off between runs**. The recurring architecture — and its footguns — live here. **Lean
index; read the linked reference for detail.**

## The common loop
1. Load persisted state (cursor + frozen filters + first-run timestamp); bootstrap on first run.
2. Pull pages at `size=1000`, replaying the **same filters** (including `from`) each call,
   until a page comes back with **fewer items than `size`**.
3. Ship each page; advance the persisted cursor **only after** a page fully ships.
4. Persist the final cursor (from that last short page) for the next run.

## Topics
- **State persistence per platform** (KV store / artifact store / control table / blob) →
  [`references/state-persistence-per-platform.md`](references/state-persistence-per-platform.md)
- **Titan non-stream replay trap** (endpoints with no cursor) →
  [`references/titan-nonstream-replay-trap.md`](references/titan-nonstream-replay-trap.md)
- **First-run bootstrap** (start-date defaults, epoch-ms detection) →
  [`references/first-run-bootstrap.md`](references/first-run-bootstrap.md)
- **Page atomicity & retry** →
  [`references/page-atomicity-and-retry.md`](references/page-atomicity-and-retry.md)

Cursor mechanics themselves are in the `intel471-api-patterns` skill
([`verity-pagination`](../intel471-api-patterns/references/verity-pagination.md),
[`titan-pagination`](../intel471-api-patterns/references/titan-pagination.md)).

## How to extend this skill
Add `references/<topic>.md`, link it above, open a pull request. When you add a new platform,
add a row to `state-persistence-per-platform.md` rather than starting a new file.

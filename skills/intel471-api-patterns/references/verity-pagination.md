# Verity471 API — Cursor Pagination

The Verity471 API paginates its stream endpoints (indicators, alerts, vulnerability
reports, `/events/stream`, etc.) with an **opaque cursor**. There is exactly one paging
mechanism — no `count`/`offset`, no timestamp workarounds. (Those belong to Titan.)

There are two ways to consume a stream, and they share the entire cursor loop below — they
differ only in whether you bound the far end with `until`, and in what you do once the stream
is drained:

- **Continuous acquisition** (the common case — a connector polling on a timer). No `until`.
  When the stream drains you keep the last cursor and resume from it next cycle. This is the
  **recommended access pattern for every Verity471 stream endpoint** used this way; follow it
  and you won't miss new or updated objects.
- **Bounded window** (a one-shot pull of a fixed period — "everything from Jan 12 to Jan 14").
  Set `from` **and** `until`, page through with the same cursor loop, and when the stream
  drains you are done: the window is fully fetched, and you discard the cursor rather than
  resuming.

## The field is `cursor_next`

Every response carries a `cursor_next` token. Pass it back as the `cursor` argument on the
next call to get the following page.

> ⚠️ **Do not confuse `cursor_next` with `cursorNext`.** The camelCase `cursorNext` is the
> **Titan** API's stream field. Verity471's SDK uses snake_case `cursor_next`. Mixing them
> up is a common copy-paste bug when porting code between the two backends.

## How to page

1. **Pick your filters once.** Optionally set `from` (SDK: `var_from`) — a 13-digit epoch-ms
   timestamp that is the start point. **Omit it entirely if you want every object in the
   stream**, from the beginning. For a bounded window, add `until` (also epoch-ms, exclusive);
   for continuous acquisition, leave `until` unset. Set `size` to the maximum, 1000.
2. **First call ever:** omit `cursor`.
3. **Each subsequent call:** send `cursor = <previous response's cursor_next>`, plus the
   **same `from` and the same everything else**, verbatim.
4. **Stop when the page is short** — fewer objects than the `size` you asked for. At that
   point the stream is drained. (A drained page can come back empty, `null`, or with the
   items key **missing from the response altogether** — read the field defensively, all three
   are just "short".)
5. **Continuous acquisition: persist that last `cursor_next` anyway** and use it as the
   `cursor` for the first request of the next acquisition cycle (an hour later, say). That is
   what walks the stream forward. **Bounded window: you're finished** — the window is fully
   fetched and the cursor has no further use.

```python
SIZE = 1000
filters = {"threat_type": "malware", "var_from": 1785542400000}  # frozen for the life of the stream
cursor = load_cursor()                                           # None on the very first run

while True:
    page = {"cursor": cursor} if cursor else {}                  # omit `cursor` on the first call
    resp = api.get_indicators_stream(**filters, size=SIZE, **page)
    items = getattr(resp, "indicators", None) or []              # empty, null, OR key absent
    process(items)
    cursor = resp.cursor_next                                    # persist after the page ships
    save_cursor(cursor)
    if len(items) < SIZE:
        break                                                    # drained — resume here next cycle
```

> On a raw dict response rather than an SDK model, use `resp.get("indicators")` instead of
> `getattr(...)` — same idea: treat *absent*, `null`, and empty alike.

The same thing over raw HTTP — note that only `cursor` changes between the two requests:

```
GET /integrations/indicators/v1/indicators/stream?threat_type=malware&size=1000&from=1785542400000
GET /integrations/indicators/v1/indicators/stream?threat_type=malware&size=1000&from=1785542400000&cursor=NTg2ZDYwOWUtNzhmMS00MDY5LTg3M2QtYTI5MWRjNzBhNTYyOjE3ODU1NjQ3ODIzNTY6ZWVkODFjMDNkMDIwYmM5ZWZmMmMwM2Q1YjM3OTZjMjczZjgyNTY5ZQ==
```

## Known traps

- **`from` is frozen, not a watermark.** Once set, `from` never changes again — not between
  pages, not between acquisition cycles. Advancing it to the last-seen timestamp each cycle is
  the cursor-less **Titan** pattern (`siem-connector-patterns` →
  [`titan-nonstream-replay-trap`](../../siem-connector-patterns/references/titan-nonstream-replay-trap.md));
  applying it to a Verity stream on top of a cursor will skip or duplicate objects. The cursor
  is the only thing that moves.

- **`until` decides which mode you're in — don't mix them up.** Setting it closes the window,
  so the cursor stops at the boundary and the stream can never be resumed forward. That's
  exactly right for a bounded backfill and wrong for continuous acquisition: a connector that
  sets `until` (to "now", say) on its first run goes permanently quiet after it drains that
  window. If you intend to keep following the stream, leave `until` unset.

- **Store the cursor from the final short page.** The terminal response still returns a
  meaningful `cursor_next` — that token is your resume point for the next run. Persist it;
  don't throw it away because the page was short or the items list dropped out of the response.

- **Filters must be replayed verbatim.** The cursor is only valid against the exact filter
  set it was issued for. Freeze the original filters alongside the cursor and send them on
  every call. To change filters, start over with no cursor.

- **The cursor is opaque.** Treat it as a black box — you can't parse it, look it up, or
  reconstruct it, and it changes every call. Persist it as an opaque string keyed by a
  stable identity (e.g. the job/stream name), not by the cursor value itself.

- **Page-level atomicity for shippers.** If you forward events downstream, advance the
  stored cursor only after an entire page has been successfully shipped. A mid-page failure
  should abort with the last fully-shipped cursor so the next run replays cleanly rather
  than skipping events.

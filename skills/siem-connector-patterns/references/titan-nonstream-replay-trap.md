# Trap: Titan non-stream endpoints have no cursor

Some Titan endpoints used by connectors are **non-stream** and expose **no `cursorNext`** —
e.g. `/v1/breachAlerts` and `/v1/cve/reports`. You page them by **timestamp**, and the naive
approach skips or duplicates records across scheduled runs.

## The trap
If you resume the next run from `lastUpdatedFrom = <last seen timestamp>`, you re-fetch every
record sharing that timestamp (dupes). If you resume from `<last timestamp> + 1`, you skip any
record that shared the boundary timestamp but hadn't been fetched yet (gaps).

## The pattern that works
1. Sort ascending (`sort: "earliest"`), page with `count` + `offset`.
2. Advance the stored watermark to **last-seen-timestamp + 1** between runs...
3. ...**and** handle the boundary within a run: keep paging by `offset` while timestamps are
   equal, so you drain all records at the boundary timestamp before advancing the watermark.
   (This mirrors Titan's own "paging beyond the 1100 offset cap" workaround — see
   `intel471-api-patterns` → [`titan-pagination`](../../intel471-api-patterns/references/titan-pagination.md).)

## Worked examples
- **Breach alerts:** the watermark is the last item's `activity.first` value, plus 1.
- **CVE reports:** walk `offset` to 1000, then advance the timestamp and restart the offset
  sequence from 0.
- **Vulnerability intelligence:** `lastUpdatedFrom = <last_updated> + 1`, `sort: "earliest"`,
  `count: 100`.

Prefer a **stream** endpoint (cursor-based) whenever one exists for the data you need — it
avoids this entirely.

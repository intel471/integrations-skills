# Titan API — Pagination

Titan has **three** paging styles. (Verity471 has only one — cursor — see
[`verity-pagination.md`](verity-pagination.md).) Pick by endpoint.

## 1. Stream endpoints — cursor
Endpoints like `/v1/indicators/stream`, `/v1/events/stream`, `/v1/credentials/stream`.
Every response carries a **`cursorNext`** field; pass it back as the `cursor` query param on
the next call, keeping **all other params identical**. Keep calling until the items list stops
coming back.

```python
resp = titan_client.IndicatorsApi(api_client).indicators_stream_get(last_updated_from=1656809200000)
# resp.cursor_next -> "MTY1NT1", resp.indicators -> [...]
resp = titan_client.IndicatorsApi(api_client).indicators_stream_get(last_updated_from=1656809200000, cursor="MTY1NT1")
# final page: resp.cursor_next -> "MTY1NT3", resp.indicators -> None
```

> **`cursorNext` vs `cursor_next`.** The raw JSON field is camelCase **`cursorNext`** (Titan).
> The Python SDK attribute is snake_case **`cursor_next`** (same as Verity471's SDK). Don't
> mix the raw and SDK forms when porting. The terminal page returns a cursor but
> `indicators = None`.

## 2. Non-stream endpoints — `count` + `offset`
One page carries up to **100 records** (`count`), and you can page up to **11 pages** (max
**`offset` = 1000**) per query — a hard **1100-result cap** per filter set.

```python
titan_client.ReportsApi(api_client).reports_get(count=100, offset=100)
```

## 3. Paging beyond 1100 — timestamp-shift workaround
When a filter set has >1100 objects, sort by `latest`, page to `offset=1000`, take the
**`created`** timestamp of the last item, then set `until=<that timestamp>` and restart the
offset sequence:

```
GET /v1/reports?sort=latest&count=100&offset=1000   -> last item created=1661864268000
GET /v1/reports?sort=latest&until=1661864268000&count=100
GET /v1/reports?sort=latest&until=1661864268000&count=100&offset=100 ...
```

Edge case: objects sharing the boundary timestamp adjacent to the last row can be missed —
for high-volume/fast-changing data (malware indicators/events, creds) use the **stream**
endpoints (style 1) instead.

## Trap: `/alerts` offset is a UID, not an integer
The `/v1/alerts` endpoint is the exception. Its **`offset` must be the `uid` of the most
recently acquired alert** (a string), not a numeric shift:

```python
titan_client.AlertsApi(api_client).alerts_get(count=100, offset="abc456")
```

## Trap: array params aren't supported in the Python client
Repeated same-name query params are AND-combined server-side, but the Python client can't send
them. Use Elastic query-string syntax in a single param instead:
`reports_get(report="(sources OR abba) AND -creaba")`.

Reference: [`titan_client/api/alerts_api.py`](https://github.com/intel471/titan-client-python/blob/main/titan_client/api/alerts_api.py)
(the uid-offset note is in the `offset` parameter docstring),
[`titan_client/configuration.py`](https://github.com/intel471/titan-client-python/blob/main/titan_client/configuration.py)
(the API-intro text embedded there covers paging limits, `/alerts`, streams and the
array/query-string rule).

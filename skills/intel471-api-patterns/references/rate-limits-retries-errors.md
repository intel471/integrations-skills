# Retries, errors, and API integration rules

## Retries
Both SDKs expose `Configuration.retries` (default `None` → urllib3's default of 3). The
Verity471 SDK accepts an `int` or a urllib3 `Retry` object.

## Rate limiting (429)
**Neither SDK gives 429 special treatment.** In the Verity471 SDK the status-specific
exceptions below stop at 422 and `500..599` maps to `ServiceException`, so a **429 falls
through to the bare `ApiException`**. Handle it yourself:

```python
from verity471.exceptions import ApiException

try:
    resp = api.get_indicators_stream(**filters, size=1000)
except ApiException as exc:
    if exc.status == 429:                       # rate limited — back off and retry
        wait = int((exc.headers or {}).get("Retry-After", 5))
        ...
    raise
```

- **429 is retryable.** Back off and retry, honouring `Retry-After` if present. Do **not** lump
  it in with the other 4xx and fast-fail — see `siem-connector-patterns` →
  [`page-atomicity-and-retry`](../../siem-connector-patterns/references/page-atomicity-and-retry.md).
- Treat sustained 429s as a signal to lower your request rate or page size, not to retry harder.
- The urllib3 `Retry` object accepted by `Configuration.retries` can retry 429 for you via
  `status_forcelist`, which is usually simpler than hand-rolling the loop.

## Verity471 typed exceptions
The Verity471 SDK raises status-specific exceptions (subclasses of `ApiException` /
`OpenApiException`):

| Status | Exception |
|---|---|
| 400 | `BadRequestException` |
| 401 | `UnauthorizedException` |
| 403 | `ForbiddenException` |
| 404 | `NotFoundException` |
| 409 | `ConflictException` |
| 422 | `UnprocessableEntityException` |
| 5xx | `ServiceException` |

Catch the specific type where you can (e.g. `UnauthorizedException` → re-check credentials).

## API integration rules
- **CORS is not allowed** — no direct browser/AJAX calls. Use a **server-side proxy**.
- **Don't cache API data long-term** — Intel 471 continually improves the data set; refetch
  rather than persist stale copies.
- **Always pin the API version.** The Python client automatically appends a `v` param equal to
  the client's package version; omitting it calls "latest" and can shift response shape.

Reference: [`verity471/exceptions.py`](https://github.com/intel471/verity471-python/blob/main/verity471/exceptions.py),
[`titan_client/configuration.py`](https://github.com/intel471/titan-client-python/blob/main/titan_client/configuration.py)
(CORS, versioning and caching guidance are in the API-intro text embedded there).

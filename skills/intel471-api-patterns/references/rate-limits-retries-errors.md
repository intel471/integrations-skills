# Retries, errors, and API integration rules

## Retries
Both SDKs expose `Configuration.retries` (default `None` → urllib3's default of 3). The
Verity471 SDK accepts an `int` or a urllib3 `Retry` object.

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

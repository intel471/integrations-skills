# The User-Agent pattern

## Token format
```
<YourOrg>-<Integration>/<version>
```
- `<YourOrg>` — a stable prefix identifying you as the publisher, so all your integration
  traffic can be attributed with one filter. Intel 471's own first-party integrations use
  `Intel471-`.
- `<Integration>` — a stable, descriptive name for the integration. Name the thing you're
  integrating *with*, so it's recognisable in a log line (e.g. `Splunk-TA`, `SentinelPlaybook`,
  `ServiceNow-App`).
- `<version>` — the integration's own version, single-sourced from the package
  (`_version.py` / `__version__` / build-time release tag), **not** hand-typed. A version that
  has to be updated by hand will be wrong within two releases.

## Setting it — Titan/Verity471 Python SDK
**Append, never overwrite** — preserve the generator token the SDK already sets
(`OpenAPI-Generator/<gen-ver>/python`), which is useful server-side:

```python
api_client = verity471.ApiClient(configuration)
api_client.user_agent = f"{api_client.user_agent}; Acme-Splunk-TA/{__version__}"
```

Works identically for `titan_client.ApiClient`.

## Setting it — raw HTTP (requests / aiohttp / JS / Logic Apps)
When you're not going through the SDK, set the header explicitly:

```python
session.headers["User-Agent"] = f"Acme-Splunk-TA/{__version__}"   # requests
```
```javascript
restMessage.setRequestHeader("User-Agent", `Acme-ServiceNow-App/${version}`);  // ServiceNow
```
For Sentinel Logic Apps, set the `User-Agent` header on the HTTP action.

## Pitfalls
- **Overwriting instead of appending** on the SDK path. You lose the generator token, and
  anyone debugging server-side can no longer tell which SDK version you're on.
- **Two names for one integration.** If a connector makes some calls through the SDK and some
  through a raw HTTP client, it's easy to end up with two different tokens. Define the string
  once, in one module, and import it.
- **A hand-typed version.** Read it from the package.

## Enforce it with a test
```python
def test_client_appends_user_agent():
    client = new_api_client(...)
    assert client.user_agent.startswith("OpenAPI-Generator/")     # SDK token preserved
    assert f"Acme-Splunk-TA/{__version__}" in client.user_agent
```

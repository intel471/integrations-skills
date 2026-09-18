# Intel 471 APIs — Auth mechanism & hosts

This covers *how* auth works and *which host* to hit. Credentials are issued from the Intel 471
Developer Portal (<https://developer.intel471.com/>).

## Both backends use HTTP Basic auth

| Surface | Basic username | Basic password | Host |
|---|---|---|---|
| **Titan** | account **email** | **API key** | `https://api.intel471.com/v1` |
| **Verity471 SDK** | "Client ID" | "Client Secret" | `https://api.intel471.cloud` |
| **Verity471 search API** | "Client ID" | "Client Secret" | `https://api.verity.intel471.com` |

```python
# Titan
configuration = titan_client.Configuration(username="<EMAIL>", password="<API KEY>")
with titan_client.ApiClient(configuration) as api_client:
    ...

# Verity471 — client id/secret are just the Basic username/password
configuration = verity471.Configuration(username="<CLIENT ID>", password="<CLIENT SECRET>")
```

## Traps
- **Verity's "Client ID / Client Secret" are just Basic username/password.** There is **no**
  `client_id`/`client_secret` kwarg on the SDK — pass them as `username`/`password`. The names
  look OAuth-shaped, but the wire protocol is plain HTTP Basic.
- **Three different hosts** for three surfaces (`api.intel471.com`, `api.intel471.cloud`,
  `api.verity.intel471.com`). Copying a base URL between projects silently points at the wrong
  backend.
- **Titan: email is the login, API key is the password** — reversing them is a classic 401.
- **Neither SDK loads credentials from the environment.** `verity471/configuration.py` never
  reads `os.environ`. Expecting a `VERITY_CLIENT_ID` env var to "just work" will fail — read
  your own config and pass the values to `Configuration(...)` yourself.
- **A config field named for one backend may hold the other's secret.** If your integration
  supports both backends behind one pair of credential fields, the *name* (`api_username` /
  `api_key`) won't tell you which backend's secret is in it. Key the meaning off the selected
  backend, not off the field name.

Reference: [`titan_client/configuration.py`](https://github.com/intel471/titan-client-python/blob/main/titan_client/configuration.py),
[`verity471/configuration.py`](https://github.com/intel471/verity471-python/blob/main/verity471/configuration.py).

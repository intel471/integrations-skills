# Porting code between Titan and Verity471

Titan and Verity471 cover overlapping data but their SDKs/endpoints differ in ways that break
copy-pasted code. Confirm the backend first.

## Method naming
- **Titan:** resource-first, `_get` suffix — `ActorsApi.actors_get`,
  `IndicatorsApi.indicators_stream_get`, `CredentialsApi.credential_sets_get`.
- **Verity471:** verb-first, `get_` prefix, usually `_stream` — `ActorsApi.get_actors_stream`,
  `IndicatorsApi.get_indicators_stream`, `CredentialsApi.get_credential_sets_stream`.

| Titan SDK | Verity471 SDK |
|---|---|
| `ActorsApi.actors_get` | `ActorsApi.get_actors_stream` |
| `IndicatorsApi.indicators_stream_get` | `IndicatorsApi.get_indicators_stream` |
| `CredentialsApi.credential_sets_stream_get` | `CredentialsApi.get_credential_sets_stream` |
| `EventsApi.events_stream_get` | `EventsApi.get_events_stream` |
| `VulnerabilitiesApi.cve_reports_get` | `ReportsApi.get_reports_vulnerability_stream` |

## Endpoint shape
- **Titan:** `/v1/<resource>` and `/v1/<resource>/stream` (e.g. `/v1/indicators/stream`).
- **Verity471:** module-prefixed, per-module version — `/integrations/<module>/v1/<resource>/stream`
  (e.g. `/integrations/creds/v1/credential-sets/stream`,
  `/integrations/intel-report/v1/reports/vulnerability/stream`).

## Params
- **Titan:** `count` (≤100) / `offset` (≤1000) / `sort` (`relevance|latest|earliest`); string
  or epoch time ranges.
- **Verity471:** `size` (1–1000, **default 1000**) / `cursor` / `var_from` (epoch-ms); **no
  sort** — streams are **oldest-first only**. To get the newest N, use the SDK helper
  [`verity471/helpers/stream_latest.py::get_latest(...)`](https://github.com/intel471/verity471-python/blob/main/verity471/helpers/stream_latest.py).

## Models & fields
- **Model naming:** Titan models are `*Schema` (`IocSchema`, `CredentialSchema`); Verity471
  models are `*Response` / `*Stream` (`IndicatorsStream`, `GetCredResponse`).
- **Field names:** both Python clients normalize API camelCase to **snake_case**
  (`documentType` → `document_type`). (The Titan API doc text mislabels this as "camel_case" —
  it means snake_case.)

## Gotcha
Several distinct Titan endpoints collapse onto one Verity stream — e.g. both
`/credentialSets` and `/credentialSets/stream` map to `get_credential_sets_stream`. Don't
assume a 1:1 endpoint correspondence.

Reference: both SDK READMEs — <https://github.com/intel471/titan-client-python>,
<https://github.com/intel471/verity471-python>.

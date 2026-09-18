# Intel 471 APIs — Date/time filter formats

Time filters differ by backend. Getting the format wrong usually returns **0 results
silently**, not an error.

## Titan — string range OR epoch
`_from` / `until` / `last_updated_from` are **strings**: "Long unix time or string time range".
Accepts relative ranges (`'2day'`, `'1day'`) or a unix timestamp. Titan stream endpoints take
epoch **milliseconds** for `lastUpdatedFrom` (e.g. `1655809200000`).

## Verity471 — strict epoch milliseconds
`var_from` / `until` are **`StrictInt`**, documented as *"Long unix timestamp in
milliseconds"* (example `1627776000000`). `from` is renamed **`var_from`** because `from` is a
Python keyword. `var_from` is inclusive; `until` is exclusive.

`until` on a stream endpoint closes the window: the cursor stops at the boundary and the stream
can't be resumed forward. Correct for a **bounded one-shot pull** ("everything from Jan 12 to
Jan 14"), wrong for **continuous acquisition** — leave it unset if you intend to keep following
the stream. See [`verity-pagination.md`](verity-pagination.md).

```python
verity471.IndicatorsApi(api_client).get_indicators_stream(var_from=1627776000000)
```

## Traps
- **Milliseconds, not seconds**, on every Verity surface. A seconds value (10 digits) lands in
  1970 and returns nothing.
- **Verity search API: hyphenated ISO dates silently return 0 results.** In the search query
  syntax `2023-01-01` is parsed with `-` as an operator, so the query means something other
  than the date you intended. Use epoch-ms (or a bare year) instead.

Reference: [`verity471/api/indicators_api.py`](https://github.com/intel471/verity471-python/blob/main/verity471/api/indicators_api.py),
[`titan_client/configuration.py`](https://github.com/intel471/titan-client-python/blob/main/titan_client/configuration.py)
(the API-intro text embedded there documents Titan's accepted time formats).

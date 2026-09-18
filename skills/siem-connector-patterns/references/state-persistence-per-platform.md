# State persistence per platform

Same idea everywhere: persist a **cursor** (where the stream left off) plus the **frozen
filters** and a **first-run timestamp**, so the next scheduled run resumes cleanly. Only the
store differs.

| Platform | Store | Cursor API / shape |
|---|---|---|
| **Splunk** (modular input) | KV store checkpoints | `helper.get_check_point(key)` / `helper.save_check_point(key, val)`; plus a separate start-time checkpoint |
| **Rapid7 InsightConnect** | `rapid7/global_artifact` | JSON state blob `{cursor, filters, initialized_at}` passed as a **string** input/output; Look Up → Delete → Add round-trip |
| **ServiceNow** | a control/state table | read cursor + initial date on entry; send `cursor` + `lastUpdatedFrom` query params |
| **Microsoft Sentinel** (Logic Apps) | Blob storage container | cursor persisted in a blob between runs |
| **Anything else** | any durable KV | one opaque string per stream, plus the frozen filter set |

## Choosing the checkpoint key
Key the cursor by the **full identity of the stream you're following**, not just the data type.
At minimum: input/job name + account + backend. Two inputs on the same data type with different
accounts or different filters are different streams and must not share a cursor.

## Notes & traps
- **The cursor field name flips by backend.** On Titan the response key is `cursorNext`; on
  Verity471 it is `cursor_next`. A connector that supports both has to branch on the selected
  backend when reading the cursor out of the response. Same trap as the API-patterns
  [`cursorNext` vs `cursor_next`](../../intel471-api-patterns/references/verity-pagination.md)
  note.
- **Store the terminal cursor.** The final short page still returns a meaningful cursor — persist
  it as the next run's resume point (see
  [`verity-pagination`](../../intel471-api-patterns/references/verity-pagination.md)).
- **The look-back start is frozen too.** `from` / `lastUpdatedFrom` is set once at bootstrap and
  replayed unchanged forever; only the cursor advances. Don't reset it to the last-seen
  timestamp each run — that's the cursor-less Titan pattern, see
  [`titan-nonstream-replay-trap`](titan-nonstream-replay-trap.md).
- **Freeze the filters with the cursor.** A cursor is only valid for the exact filter set it was
  issued against; persist filters alongside and replay them verbatim.
- **Watch out for platforms that destructure your state.** On Rapid7 InsightConnect the `state`
  input must be a plain JSON **string**; if you model it as a structured object the Builder
  splits it into separate fields, state doesn't round-trip, and the connector silently
  re-bootstraps on every run — producing duplicate events forever with no error. Whatever the
  platform, assert in a test that state written on run N is byte-identical when read on run N+1.

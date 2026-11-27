# Plaintext passwords in compromised credentials

Compromised-credential records can carry the leaked password **in plaintext**. Handle it as the
most sensitive value a connector will ever touch: store it only when the user has explicitly
asked for it, and keep every other copy out of logs, history and raw-payload storage.

## Where it appears (Verity471)

| Stream | Field |
|---|---|
| `/integrations/creds/v1/credentials/stream` | `.credentials[].data.password.password_plain` |
| `/integrations/creds/v1/credentials/occurrences/stream` | `.credential_occurrences[].data.credential.password.password_plain` |

The rest of the `password` object (strength, length, complexity metadata) is not sensitive in
the same way and can be mapped normally.

## It is entitlement-gated, per affiliation group

Whether `password_plain` is returned is an **account setting, per affiliation group**
(`my_employees`, `my_customers`, `third_parties`, `vip_emails`). It is **separate from having the
affiliation group itself enabled and configured**: an account can have `my_customers` fully
set up and still have plaintext passwords disabled on it.

Consequences for connector code:
- **A missing `password_plain` is normal, not a bug.** It has two causes: the account isn't
  entitled to plaintext for that record's affiliation group, or the plaintext simply isn't in
  Intel 471's data for that credential. **Not every credential has a plaintext password, even
  with the entitlement on.** Treat the field as optional; never fail a page over its absence.
- **One stream can mix both.** Records from the same page can differ, whether because plaintext
  is enabled for `my_employees` but not `my_customers` or because some credentials have no
  plaintext at all. Don't infer the entitlement from any single record, and don't read an
  absent field as "entitlement off".
- If a user expects plaintext and isn't getting it, it's an account setting on the Intel 471
  side, not something the connector can switch on.

## Convention: an explicit opt-in toggle, off by default

Every connector that ingests the credentials streams should expose a setting such as
**"Store plaintext passwords"** on the credentials stream configuration, **off by default**.

- **Off:** remove `password_plain` from every record before anything is persisted or logged.
- **On:** store it in the mapped record, and nowhere else (see the trap below).
- The toggle only controls *storage*. It can't make the API return plaintext the account isn't
  entitled to, or that isn't in the data for a given credential. The UI or docs for the setting
  should say so, so users don't read an empty field as a broken toggle.

## Trap: the mapped record is not the only copy

Dropping the field from the mapped output isn't enough. Integrations routinely keep the raw API
response in places nobody thinks of as storage, and each one is a plaintext-password leak:

- **Request/response history** kept by the platform: workflow or run history (e.g. Azure Logic
  Apps run history; mark the HTTP action's inputs and outputs as secure), job or step logs,
  integration debug logs.
- **The raw payload stored next to the mapped data**: a `_raw` field, an "original event" or
  "raw JSON" column, an attachment of the source object.
- **HTTP client debug logging** and exception messages that include the response body.
- **Retry, dead-letter and error queues** that hold a failed page verbatim.
- **Test fixtures** captured from a real response.

**Strip early.** Remove `password_plain` right after deserialisation, before the record reaches
any logger, raw-payload field or queue, unless the toggle is on. Even when it's on, the
mapped field should be the **only** place the value lands: raw-payload copies and logs should
still have it removed or masked.

---
name: user-agent
description: How an Intel 471 integration should identify itself and its version in the User-Agent header it sends to the Titan/Verity471 APIs, and how to set it per stack (Python SDK, raw HTTP, JS/Logic Apps). Consult when creating a new integration, adding an API client, or reviewing an integration's connection setup.
---

# Identifying your integration — the User-Agent convention

Every integration should identify itself and its version in the `User-Agent` it sends to the
Intel 471 APIs.

Why it's worth the five lines of code:

- **Support can see you.** When you open a ticket about odd paging behaviour, a named,
  versioned User-Agent lets Intel 471 find your traffic instead of asking you to reproduce it.
- **You find out before your users do.** If a release of yours starts erroring or hammering an
  endpoint, it is attributable to a version.
- **Anonymous SDK traffic is indistinguishable.** Without it you are one of many callers sending
  the bare `OpenAPI-Generator/<ver>/python` default.

## Topics
- **The pattern** (token format + how to set it per stack) →
  [`references/pattern.md`](references/pattern.md)

## TL;DR
- Send `<YourOrg>-<Integration>/<version>` (Intel 471's own first-party integrations use the
  `Intel471-` prefix, e.g. `Intel471-Rapid7-Logshipper/1.4.0`).
- With the Titan/Verity471 **SDK**, **append** to the SDK's existing user agent — don't
  overwrite it.
- Single-source `<version>` from the package version; never hand-type it.
- Add a test that asserts the token is present, so it can't silently regress.

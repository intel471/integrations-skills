# Intel 471 integration skills

This repository is a set of **skills for coding agents**: field-tested patterns and traps for
building integrations against the Intel 471 **Titan** and **Verity471** APIs. It contains no
runnable code — it is knowledge, structured so an agent loads only the part it needs.

If you are an agent working on code that calls the Intel 471 APIs, read the relevant skill
below **before** writing pagination, auth, or scheduling logic. These documents exist because
each trap listed in them has already cost someone a debugging session.

## Available skills

| Skill | Read it when |
|---|---|
| [`skills/intel471-api-patterns`](skills/intel471-api-patterns/SKILL.md) | Writing or debugging any call to the Titan or Verity471 API/SDK — especially anything that pages through results (`cursor` / `cursor_next` / `count`+`offset`), authenticates, or filters by time. |
| [`skills/siem-connector-patterns`](skills/siem-connector-patterns/SKILL.md) | Building or debugging a **scheduled** connector that pulls a stream on a timer and ships it somewhere (SIEM, data lake, ticketing) and must remember where it left off between runs. |
| [`skills/user-agent`](skills/user-agent/SKILL.md) | Creating a new integration or adding an API client — so your traffic is identifiable and versioned. |

Each skill is a lean `SKILL.md` index plus a `references/` directory. **Read the index first,
then open only the reference file for the topic at hand** — the references are written to be
read individually.

## The five traps that cause the most damage

If you read nothing else:

1. **Verity471 `from` is frozen, not a watermark.** Set it once; only the cursor moves. Advancing
   `from` each cycle on top of a cursor skips or duplicates objects.
   → [`verity-pagination`](skills/intel471-api-patterns/references/verity-pagination.md)
2. **`cursorNext` (Titan, raw JSON) vs `cursor_next` (both Python SDKs).** The single most common
   copy-paste bug when porting between backends.
   → [`titan-vs-verity-porting`](skills/intel471-api-patterns/references/titan-vs-verity-porting.md)
3. **Epoch milliseconds, not seconds.** A 10-digit timestamp lands in 1970 and silently returns
   nothing — no error.
   → [`date-time-handling`](skills/intel471-api-patterns/references/date-time-handling.md)
4. **Setting `until` on a stream closes it permanently.** Right for a bounded backfill; it makes
   a continuous connector go quiet forever after its first drain.
   → [`verity-pagination`](skills/intel471-api-patterns/references/verity-pagination.md)
5. **Titan's `/v1/alerts` `offset` is a record `uid` string, not an integer.**
   → [`titan-pagination`](skills/intel471-api-patterns/references/titan-pagination.md)

## Using these skills in your own project

The skills are plain Markdown, so any agent can read them. Pick whichever fits your tooling —
see [README.md](README.md#use-it-with-your-agent) for the per-agent instructions:

- **Claude Code** — install as a plugin (one command, auto-loads in every project).
- **Any agent that reads `AGENTS.md`** (OpenAI Codex, Cursor, Gemini CLI, Jules, Zed, Aider,
  Factory, Devin) — vendor the `skills/` directory into your repo and link it from your own
  `AGENTS.md`.
- **Cursor / GitHub Copilot / Windsurf** — pre-generated rule files are in `.cursor/rules/`,
  `.github/instructions/` and `.windsurf/rules/`; copy them into your repo.

## Conventions for contributors

- **One fact per reference file, indexed from `SKILL.md`.** Short index, deep references.
- **Write the `description` frontmatter for retrieval** — it decides whether an agent loads the
  skill at all. Name the backend, the API and the task.
- **Capture the trap, not just the happy path.** The gotchas are the highest-value content.
- **Cite an openly readable source.** Link the public SDK on GitHub, not a private repo path.

See [CONTRIBUTING.md](CONTRIBUTING.md).

# Intel 471 Integration Skills

**Field-tested patterns and traps for building integrations against the Intel 471 APIs**
— packaged as skills that your coding agent loads on demand.

If you build a connector, script, or app against the Intel 471 **Titan** or **Verity471**
APIs, this repository gives your agent the knowledge that normally only comes from having
already shipped one: how the cursor actually works, which filters silently return zero
results, and what breaks when you copy code between the two backends.

Agent-agnostic and plain Markdown. Works with Claude Code, OpenAI Codex, Cursor, GitHub
Copilot, Gemini CLI, Windsurf, and anything else that reads files.

---

## Why bother

Both Intel 471 backends have failure modes that produce **no error** — just wrong results you
won't notice for a week:

- A 10-digit (seconds) timestamp instead of 13-digit (milliseconds) silently lands in 1970 —
  pulling **your entire history** on a lower bound, or **zero records** on an upper bound.
- Advancing the `from` filter each cycle on top of a cursor **skips or duplicates** objects.
- Setting `until` on a stream makes a continuous connector **go permanently quiet** after its
  first drain.
- `/v1/alerts` takes a record **uid string** as its `offset`, not an integer.

Each of those is one line in a reference file here. Your agent reads the line instead of
reproducing the bug.

## What's in here

| Skill | Covers |
|---|---|
| **[`intel471-api-patterns`](skills/intel471-api-patterns/SKILL.md)** | Pagination (all three Titan styles + Verity471 cursors), auth and hosts, date/time filters, retries and typed exceptions, and the differences that break code ported between backends. |
| **[`siem-connector-patterns`](skills/siem-connector-patterns/SKILL.md)** | Architecture for a **scheduled** connector: persisting a cursor and frozen filters between runs, drain-until-short-page pulls, first-run bootstrap, page-level atomicity, and the Titan non-stream replay trap. |
| **[`user-agent`](skills/user-agent/SKILL.md)** | Identifying your integration and its version to the API, per stack. |

Start with **[AGENTS.md](AGENTS.md)** for the guided tour, including the five traps that cause
the most damage.

### How a skill is shaped

```
skills/intel471-api-patterns/
├── SKILL.md              # lean index — this is all that stays loaded
└── references/           # the detail — loaded only when relevant
    ├── verity-pagination.md
    ├── titan-pagination.md
    └── …
```

Only each skill's `name` and `description` sit in the agent's context permanently. When your
task matches a description, the agent reads `SKILL.md`, then pulls in just the one reference
file it needs. Idle knowledge costs almost nothing, so the corpus can grow without making
every session more expensive.

---

## Use it with your agent

### Claude Code

Install once; it then loads automatically in every project, with no per-project setup.

```bash
/plugin marketplace add https://github.com/intel471/integrations-skills.git
/plugin install integrations-skills@intel471
```

The agent invokes a skill on its own when your task matches it. To force it, say
"use the intel471-api-patterns skill", or pick it from the `/` menu.

To update later:

```bash
/plugin marketplace update intel471
/plugin update integrations-skills@intel471
```

### Any agent that reads `AGENTS.md`

OpenAI Codex, Cursor, Gemini CLI, Jules, Zed, Aider, Factory and Devin all read `AGENTS.md`
from your repo root. Vendor the skills into your project and point your own `AGENTS.md` at
them:

```bash
git clone --depth 1 https://github.com/intel471/integrations-skills.git /tmp/i471-skills
mkdir -p docs/intel471-skills
cp -R /tmp/i471-skills/skills/* docs/intel471-skills/
```

Then add to your `AGENTS.md`:

```markdown
## Intel 471 API integrations
Before writing or changing code that calls the Intel 471 Titan or Verity471 APIs, read
`docs/intel471-skills/intel471-api-patterns/SKILL.md`. For scheduled connectors that
persist a cursor between runs, also read
`docs/intel471-skills/siem-connector-patterns/SKILL.md`.
```

Or add this repo as a git submodule if you'd rather track upstream:

```bash
git submodule add https://github.com/intel471/integrations-skills.git vendor/intel471-skills
```

### Cursor

Pre-generated rules are in [`.cursor/rules/`](.cursor/rules). Copy them into your project;
they auto-attach when you touch matching source files.

```bash
mkdir -p .cursor/rules
cp /tmp/i471-skills/.cursor/rules/*.mdc .cursor/rules/
```

These link to the skills on GitHub, so they work as-is — no vendoring required. If you'd
rather your agent read local copies, vendor `skills/` as above and repoint the links.

### GitHub Copilot

Pre-generated instructions are in [`.github/instructions/`](.github/instructions) (path-scoped
via `applyTo`) plus a repo-wide
[`.github/copilot-instructions.md`](.github/copilot-instructions.md).

```bash
mkdir -p .github/instructions
cp /tmp/i471-skills/.github/instructions/*.md .github/instructions/
```

### Gemini CLI / Windsurf

[`GEMINI.md`](GEMINI.md) and [`.windsurf/rules/`](.windsurf/rules) are generated for the same
purpose — copy whichever your tool reads.

> **All the generated adapters link to the skills on GitHub**, not to relative paths, so you
> can drop any of them into any repository and the links resolve. Vendoring `skills/` locally
> is optional — do it if you want your agent reading local files, or if you need to pin the
> content rather than track `main`.

> **One caveat, stated plainly:** rule files like `.cursor/rules` and
> `.github/copilot-instructions.md` apply to an agent working **inside the repo that contains
> them**. Copying them into your project is what makes them take effect; they do nothing from
> here. Only the Claude Code plugin install is global to your machine.

---

## Accuracy, scope and support

- **This is knowledge, not a product.** No runnable code, no API client. For the clients, see
  [`titan-client-python`](https://github.com/intel471/titan-client-python) and
  [`verity471-python`](https://github.com/intel471/verity471-python).
- **It captures traps, not the full API surface.** Endpoint and field reference lives in the
  Intel 471 Developer Portal (<https://developer.intel471.com/>, customer login required).
- **It reflects behaviour observed at a point in time.** APIs change. If something here is
  wrong or out of date, please [open an issue](https://github.com/intel471/integrations-skills/issues)
  — corrections are the most valuable contribution.
- **Not a support channel.** For questions about your Intel 471 account, entitlements, or API
  credentials, contact Intel 471 support through your usual channel.

## Contributing

Corrections and new traps are welcome from anyone — including customers and partners. A new
pattern is a small pull request. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[Apache-2.0](LICENSE).

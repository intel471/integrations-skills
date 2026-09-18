# Contributing

Corrections and new traps are welcome from anyone — Intel 471 staff, customers and partners
alike. If you lost an afternoon to something, writing it down here saves the next person that
afternoon.

**The most valuable contribution is a correction.** These documents describe API behaviour
observed at a point in time. If something is wrong, out of date, or was never quite right,
please say so — [open an issue](https://github.com/intel471/integrations-skills/issues) even if
you don't want to write the fix yourself.

## Add a new topic to an existing skill

1. Create `skills/<skill>/references/<topic>.md` with the detail (examples, gotchas, snippets).
2. Add a one-line link to it under the right heading in that skill's `SKILL.md`.
3. Open a pull request.

## Add a whole new skill

1. Create `skills/<new-skill>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: my-skill
   description: >-
     One or two sentences on what this covers AND when an agent should reach for it.
     This is the only part that stays loaded, so make the triggers explicit —
     name the APIs, the task, and the keywords.
   ---
   ```
   The `name` must match the directory name.
2. Keep `SKILL.md` a lean index; put the detail in `references/`.
3. Run `python3 tools/sync-adapters.py` and commit the regenerated adapter files.
4. Skills under `skills/` are auto-discovered — there's no registry to update.

## Guidelines

- **One fact per reference file, indexed from `SKILL.md`.** Short index, deep references. A
  reference file should be readable on its own, because that's how it gets read.
- **Write the description for retrieval.** It decides whether an agent loads the skill at all.
  Name the backend, the API, the task and the trigger words. This is the single highest-leverage
  line in a skill.
- **Capture the trap, not just the happy path.** "It returns 0 results with no error if you
  pass seconds instead of milliseconds" is worth more than a correct example.
- **Cite an openly readable source.** Link the public SDK on GitHub or the official docs. Don't
  cite a private repository path or a line number in code a reader can't open — the citation
  exists so someone can verify the claim.
- **Say when you're unsure.** "Observed on the creds stream in 2025-08; unclear whether it
  applies to all streams" is useful. False confidence is not.

## Regenerating the per-agent adapters

The files in `.cursor/rules/`, `.github/instructions/`, `.windsurf/rules/`, plus
`.github/copilot-instructions.md` and `GEMINI.md`, are **generated** from the `SKILL.md`
frontmatter. Don't hand-edit them:

```bash
python3 tools/sync-adapters.py           # regenerate
python3 tools/sync-adapters.py --check   # CI-friendly: non-zero exit if stale
```

They contain only a pointer plus the trigger description — never a copy of a skill body — so
the knowledge itself lives in exactly one place.

## Validating

If you have Claude Code installed, check the plugin manifests still parse:

```bash
claude plugin validate .
```

(A warning about a missing `version` is expected — see below.)

## Versioning

This repository deliberately sets **no `version` field** in `plugin.json`, so Claude Code
versions it by git commit SHA. Every merge to `main` is automatically the latest for everyone,
with no version numbers to bump — appropriate for a living document. Pin to a tag or SHA
yourself if you need reproducibility.

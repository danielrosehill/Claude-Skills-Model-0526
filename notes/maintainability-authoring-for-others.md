# Maintainability — Can An Author Still Ship Plugins For Others While Running This Themselves?

*Captured 2026-05-04. Exploratory.*

A core requirement: **adopting the DB-backed model for personal use must not fork the author away from the public plugin ecosystem.** If choosing the server-side model means I can no longer ship plugins to other Claude Code users via the standard marketplace flow, the model is a non-starter — it would isolate the most prolific authors from the people they write for.

## The risk

There are two failure modes to avoid:

1. **Bifurcated authoring**: I write skills in a DB-native form (rows, embeddings, internal IDs) that no longer round-trips back to a `plugin/` folder on disk. Public users — who consume plugins via Anthropic's marketplace mechanism — can't install what I wrote.
2. **Drift between forms**: I keep both representations but the DB version becomes the "real" one and the markdown version rots. Public consumers get stale skills; my private copy keeps moving.

## The design constraint that solves it

The repo, not the database, is the source of truth for **authoring**.

- Skill authoring stays exactly as Anthropic designed it: a git repo containing `plugin.json` and `skills/<name>/SKILL.md`, plus optional `commands/`, `agents/`, `scripts/`.
- Publishing to **public users** = `git push` to a public marketplace repo. Nothing changes for them.
- Publishing to **my own runtime** = the same push, picked up by my home-server ingester (webhook or scheduled pull), which parses the repo and upserts into the DB.

The DB is **downstream of the repo**, never upstream. There is no "edit in the DB" path that bypasses the markdown form. Every record in the DB is reproducible by re-ingesting the source repo at a given commit.

## Authoring workflow, end to end

1. Edit `skills/foo/SKILL.md` in a public plugin repo on disk.
2. Test locally with Claude Code's `--plugin-dir <local path>` — same loop Anthropic documents.
3. `git push`.
4. Public users install via the marketplace, unchanged.
5. My ingester pulls the commit, parses, embeds, upserts. My next session sees the new version through the lookup MCP.

I never touch a database manually. Public users never see the database. The repo is the lingua franca.

## Private skills are a separate axis

For skills I don't want to ship publicly, the same mechanism still applies — they live in a **private** marketplace repo (or a personal repo with no marketplace at all), pushed to a private git remote, ingested into the DB with `visibility=private`. The authoring shape is identical; only the destination repo and the visibility flag differ. See `marketplaces-and-user-workspaces.md` for how the visibility tiers are enforced on lookup.

## Implication: the DB schema must be lossless for the markdown form

Whatever the schema looks like, it must be possible to **regenerate** an authoring-shaped repo from the DB. This is a sanity check, not a daily workflow — but if the DB ever loses information that the markdown form had (frontmatter quirks, comments in scripts, multi-file skill bundles, asset references), then the DB has silently become incompatible with public distribution. A round-trip test in CI is cheap insurance.

Concretely:
- Frontmatter preserved verbatim (not re-serialized).
- Body preserved verbatim (no Markdown re-rendering).
- Bundled scripts/templates preserved as blobs with their original paths.
- `plugin.json` preserved verbatim alongside the skill records.

## Implication: tooling lives in the repo, not the DB

Helper scripts for authoring — linters, frontmatter validators, dependency checkers — should run against the **repo on disk**, not the DB. That keeps them runnable by anyone who clones the repo, including users who have never heard of the DB-backed model.

## What this protects against

- **Ecosystem isolation**: I keep shipping to public users on the standard marketplace path. They install with `claude plugin marketplace add ...` exactly as before.
- **Vendor lock-in to my own infra**: if I retire the home server tomorrow, I lose the lookup-MCP convenience but nothing about the plugins themselves. The repos still exist; I (or anyone else) can install them the standard way.
- **Forking the user community**: there's no "DB-only" plugin format that splits authors from consumers. The DB is a personal runtime optimization, invisible to anyone I publish for.

## What it doesn't solve

- If Anthropic changes the plugin format substantially, my ingester has to keep up — same maintenance burden any plugin author has.
- If a public consumer has hundreds of my plugins installed, *they* still pay the context cost the model is designed to avoid. The DB-backed model doesn't help them unless they adopt it too. Which is fine — that's their choice; the public path keeps working as Anthropic designed it.

## Test for the model

A one-line check: *"After I push to my public plugin repo, does a fresh Claude Code install on a stranger's machine — using only Anthropic's documented marketplace flow — get the same plugin behavior I get through my lookup MCP?"*

If yes, the model is sound. If no, the schema or the ingester has dropped something that mattered, and the markdown form has stopped being canonical.

# Marketplace Tiers and User-Workspace Preservation

*Captured 2026-05-04. Addendum to the namespaced-progressive-discovery thoughts and the database-backed plugin server plan.*

## Two unsolved aspects to lock down

1. **Marketplace tiers**: Daniel ships through two distinct marketplaces today — a **public** one (`Claude-Code-Plugins`, intended for distribution) and a **private** one (`Claude-Code-Plugins-Private` / `Daniel-Rosehill-Claude-Plugins-Private`, for internal-only plugins). Anthropic's docs treat this as a first-class concern: marketplaces can live in public repos or private ones (`/en/plugin-marketplaces#private-repositories`). Any DB-backed model that flattens this distinction breaks the distribution story.

2. **User-workspace pattern**: Daniel's plugins consistently use the "skill writes a pointer to `$CLAUDE_USER_DATA/<plugin>/config.json`, which references a user-chosen workspace path on disk" pattern (see `declutter-genie/setup-workspace`, `image-production/setup-workspace`, `op-vault/save-to-vault`, etc.). This exists *because* plugins are distributed — the plugin author can't bake `/home/daniel/...` paths in; each user picks their own workspace location. The DB-backed model must preserve this pattern, not absorb it.

The two concerns are linked: tier matters for **distribution**; workspace matters for **runtime per-user state**.

## Anthropic's model (per the copied reference docs)

- Plugins bundle skills, agents, commands, hooks, MCP servers, and LSPs into a coherent unit (`reference/anthropic-docs-plugins.md`).
- Marketplaces are the distribution channel. A marketplace is a git repo (public or private) containing a `marketplace.json` manifest pointing to plugin definitions. Private repos are explicitly supported as the team-internal path.
- There is **no native concept of user workspace state**. Skill bodies are content the agent reads; whatever data a skill creates on the user's machine is the skill author's convention, not the runtime's. Daniel's `$CLAUDE_USER_DATA` pattern is a *user-land convention layered on top* of Anthropic's primitives.

The DB-backed model proposed in `notes/idea-database-backed-plugin-server.md` must respect both: it changes the *distribution + retrieval substrate*, not the *authoring model* and not the *per-user runtime state mechanism*.

## Marketplace tiers in the DB-backed model

Each plugin and skill record in the database carries:

| Field | Purpose |
|---|---|
| `source_marketplace` | The marketplace the plugin was ingested from (e.g. `claude-code-plugins`, `claude-code-plugins-private`). FK to a `marketplaces` table. |
| `source_repo` | The git repo URL (so re-ingest / update is traceable). |
| `visibility` | `public` / `private` / `internal`. Drives access control on the lookup MCP. |
| `owner_identity` | Authoring identity (Daniel, a team, an external author). |
| `version` | Plugin version (immutable per record; "current" pointer separate). |

The `marketplaces` table maps to **named marketplaces with their own visibility tier**. Creating a new marketplace = inserting a row with: name, source repo URL, visibility tier, ingest schedule, and any auth scoping. This mirrors what Anthropic exposes (a marketplace.json in a repo) but lifts it into the database as a first-class concept the lookup MCP can reason about.

### Lookup MCP enforces the tier

The lookup MCP service connection carries an identity (Cloudflare Access, service token, or similar). On every `list_namespaces` / `list_skills` / `search_skills` / `get_skill` call, results are filtered by `visibility` against the caller's identity:

- Anonymous / public callers see only `visibility=public` records.
- Authenticated Daniel-identity sees public + private + internal.
- Future: team identities can see public + their team's private.

This is the **same enforcement Anthropic gives you via private-repo marketplaces** (you can only install from a repo you can clone), translated into the DB-server world.

### Creating a new marketplace tier

Three axes a new marketplace might want:

1. **Public, redistributable** — sourced from a public git repo, embeddings + bodies ingested unchanged, served to any caller. (Daniel's `Claude-Code-Plugins`.)
2. **Private, single-user** — sourced from a private repo, served only to authenticated owner identity. (Daniel's existing private marketplace.)
3. **Private, team-shared** — sourced from a private repo, served to any identity in the team's allow-list. (Future expansion; not needed today but the schema should not preclude it.)

The CLI for marketplace management becomes:

```
plugins-server marketplace add  <name> --repo <url> --visibility <public|private> [--owners ...]
plugins-server marketplace ingest <name>          # walks the repo, upserts plugins/skills
plugins-server marketplace list
plugins-server marketplace remove <name>
```

That gives a clean parallel to today's `claude plugin marketplace add ...` CLI but operating on the database-backed substrate.

### Authoring loop, unchanged

Plugin authoring stays exactly as Anthropic designed it: write `plugin.json`, `skills/<name>/SKILL.md`, etc. in a repo. To publish:

- **Public**: push to the public marketplace repo. Server's ingester picks it up (webhook or scheduled pull).
- **Private**: push to the private marketplace repo. Same ingester, different visibility tag.

No author has to learn the database. The repo *is* the source of truth for editing; the database is the runtime substrate. This matches the recommendation in `anthropic-docs-plugins.md` that local `--plugin-dir` overrides marketplace versions for development — devs still iterate locally, and the central server only sees a plugin once it's pushed.

## User-workspace pattern: preserved as-is

The DB-backed model **must not** absorb the `$CLAUDE_USER_DATA` convention. The reasoning:

- The database stores **skill definitions** — the *what to do* — not user state.
- The user's workspace (their inventory file, their audio episodes folder, their op-vault metadata) is **machine-local and user-chosen**. It cannot live on a shared server, both for privacy and because the actual artifacts on disk (audio files, images, repos) need to be where the user is operating.
- Daniel's existing pattern — skill writes `$CLAUDE_USER_DATA/<plugin>/config.json` containing a pointer to the user's chosen workspace path — is precisely the right boundary, and it works *whether the skill body comes from a local plugin folder or from a database fetch*. Nothing about the source of the skill text changes how the bash blocks inside that text resolve `$CLAUDE_USER_DATA`.

### What this means in practice

- When `get_skill("declutter-genie:setup-workspace")` returns the SKILL body, the agent runs the bash inside it locally, the user picks a workspace path, the pointer lands at `$CLAUDE_USER_DATA/declutter-genie/config.json` — exactly as today.
- Subsequent skills in the same plugin (`import-inventory`, `find-duplicates`, etc.) read that pointer the same way they always did.
- The DB never sees the workspace path or any user data. The DB only sees the skill *text*, and (separately) ingestion telemetry — which plugin / version / call counts, if Daniel wants that.

### A tiny tightening worth making

Anthropic exposes a `${CLAUDE_PLUGIN_ROOT}` for plugins that need to reference bundled scripts/files (per the plugins reference). In the DB-backed model, when a skill bundles non-text artifacts (Python scripts, templates, binaries), the lookup tool needs to materialize them to a tmp dir and set `CLAUDE_PLUGIN_ROOT` to point there before the skill runs. That's an implementation detail of `get_skill` for skills that aren't pure markdown — flagged here so it's not forgotten when we get past markdown-only skills.

## What changes in the plan

Add to `notes/idea-database-backed-plugin-server.md`:

1. **Marketplaces are first-class records** in the schema, with a visibility tier per marketplace, propagating to plugins and skills sourced from them.
2. **The lookup MCP enforces visibility on every call** based on caller identity.
3. **Authoring loop is unchanged from Anthropic's model** — repos remain the source of truth; the DB is downstream of `git push`.
4. **`$CLAUDE_USER_DATA` workspace pattern is preserved verbatim** — the DB stores skill definitions, never user state. Skill bodies fetched from the DB resolve `$CLAUDE_USER_DATA` locally exactly as they do today.
5. **Bundled non-markdown plugin artifacts** (scripts, templates) must be materialized and `CLAUDE_PLUGIN_ROOT` set on retrieval, mirroring Anthropic's runtime contract.

I'll fold a condensed version of points 1–5 into the plan file in the same pass.

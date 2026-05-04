# Database-Backed Plugin/Skill Server — Rough Notes

*Captured 2026-05-03. Exploratory, not a spec.*

## The pain point

Plugins are an excellent abstraction for Claude Code — they bundle the agentic primitives (skills, MCP connections, commands, agents) into a coherent unit and give a framework for using them together. The marketplace ecosystem is already strong: add a marketplace, add a plugin, it's available.

But we're walking into a context-window problem that rhymes with the original MCP one:

- **MCP, originally**: define your servers, and at session start the tool definitions get loaded into context. Before the first user token, the window is already narrowed substantially.
- **Plugins, now**: same shape. Enable a useful set, and a meaningful slice of context is consumed at session start by skill metadata, agent definitions, and command surfaces — before any actual work begins.

## Why "just be selective" is a bad answer

The commonly suggested mitigation — *"enable plugins per-workspace, only the ones you need for this project"* — is presented as if it were free. It isn't.

The whole appeal of agentic Claude Code is **fluidity**: you drop into a terminal anywhere on the home network, you ask for something, it works. Per-workspace plugin definitions reintroduce the kind of friction that MCP-server-per-project caused — having to remember which workspace has which plugin enabled, maintaining N copies of overlapping config, losing work because you opened the wrong folder. It runs directly against the model of Claude as an ambient agent.

## The MCP analogue (already solved, for me)

For MCP I solved this by bundling on the **server side**, not the client side:

- All MCP connections aggregated on a home-server surface
- Single Cloudflare-fronted entry point
- One connection in Claude Code → access to the full toolset
- Nothing bound to the local environment; same surface from any device

Client-level config is reserved only for things that genuinely *must* be local.

## The same pattern, applied to plugins

Plugins should follow the same architectural move. The question is *how*, given that plugins today are folders of markdown that have to be read in by an agent.

The proposal: **stop treating the markdown folder as the primary representation.** Treat it as an intake/exchange format. The actual primary representation is a database, hosted centrally (home server VM, on-prem box, whatever), with skills, scripts, agents, and commands as **first-class citizens**.

### Two database shapes worth considering

**Option A — traditional relational/document store**

Mirror the current shipping structure:

```
marketplace → plugin → { skills, agents, commands, scripts, mcp-servers }
```

with secret fields gated through proper secret storage. Straightforward, easy to reason about, easy to author against.

**Option B — vector-native (or graph)**

Because the real win here is **semantic discovery on demand**, the store should ideally be vector-native from the ground up — or a graph DB with embeddings on the nodes. Even Option A should have a vectorized index layered on it; the question is whether vectors are bolt-on or load-bearing.

The graph framing is interesting because plugins already have natural edges: *plugin contains skill*, *skill depends on MCP server*, *command invokes skill*, etc. Traversal beats keyword search for "what do I have that could do X."

## The runtime model

What Claude Code actually sees:

- **One** thing registered: a `skill-lookup` (or `plugin-lookup`) MCP connection or skill.
- That single surface is the gateway to the whole library.
- At session start, context cost is ~constant regardless of how many plugins exist in the database — you're not loading 80 SKILL.md files, you're loading one tool that knows how to find them.
- On demand, the agent queries: *"I need to do X — what skills are relevant?"* The lookup returns a small ranked set, the agent pulls the full definition for the one(s) it wants, and only then does that content enter context.
- Definitions can be **updated on use**, not on session start — the database is the source of truth, and the local cache (if any) is just a perf optimization.

This is essentially the same shape as a vector store for RAG — but the "documents" are skill definitions, and the "retriever" is the lookup tool.

## Namespaces and progressive discovery (added 2026-05-04)

Pure semantic retrieval is the wrong primary interface — it assumes the agent already knows what to ask for. The right primary interface is **namespaces**: a shallow, curated taxonomy that lets the agent narrow *before* retrieving. Vector search lives underneath, scoped by namespace when useful.

This is **not** federation (multiple servers, MCP-aggregator style). It's discovery hierarchy on a single store — the same move a filesystem makes vs. a flat blob bucket.

### Namespace as a first-class entity

Alongside `skill`, `agent`, `command`, `script`, the data model gets a `namespace` (and optionally `sub_namespace`, max two levels deep).

Each skill record carries:
- `namespace` — the discovery axis (curated taxonomy: `audio`, `business`, `israel`, `repo-ops`, `decision-frameworks`, `comms`, `infra`, …)
- `plugin_of_origin` — metadata for attribution and updates, kept separate from namespace
- optionally, multiple namespaces (skills that legitimately span)

Curate the taxonomy by hand once over the existing ~60 plugins; classify new skills at ingest time with LLM assistance + a human review pass. Quality of names is load-bearing — `misc`/`tools`/`stuff` is worse than no taxonomy.

### Four lookup tools, not two

- `list_namespaces()` → top-level categories (~20 lines of output)
- `list_skills(namespace)` → skills in that namespace, name + one-line desc only
- `search_skills(query, namespace=None)` → vector search, optionally scoped to a namespace (scoped search beats global search — embedding space is less cluttered within a domain)
- `get_skill(id)` → full body, on demand

### The agent's first-turn pattern

System prompt teaches the contract: *"Use `list_namespaces` to discover capability areas. Use `search_skills` for semantic lookup, ideally scoped to a namespace. Use `get_skill` to load a specific skill body."*

User asks "denoise this WAV." Agent calls `search_skills("denoise", namespace="audio")` → 3 hits → `get_skill("audio:denoise")` → executes. Total discovery cost: ~50 lines, vs. the thousands currently loaded at session start.

### Author contract

Skill authors must write descriptions that work at **both** levels — the one-line listing (terse, action-oriented) and the full body (operational detail). Frontmatter `description:` is the listing form; the body is the full form. Same as today, but now the listing form is what the agent sees first, so its quality matters more.

## Marketplace tiers and user workspaces (added 2026-05-04)

Grounded on Anthropic's docs in `reference/anthropic-docs-{plugins,mcp,skills}.md`. Two concerns the model must address explicitly: distribution tiers (public vs private marketplaces) and the user-workspace pattern (per-user state on the local machine).

### Marketplaces are first-class records

The schema gets a `marketplaces` table. Each plugin/skill record carries:

- `source_marketplace` (FK → marketplaces)
- `source_repo` (git URL — re-ingest path)
- `visibility` (`public` / `private` / `internal`)
- `owner_identity` (authoring identity)
- `version` (immutable per record; "current" pointer separate)

Each marketplace row has its own visibility tier, ingest source, and owner/team allow-list. This mirrors Anthropic's public-vs-private-repo marketplace distinction, lifted into the database.

### Visibility is enforced on every lookup call

The lookup MCP carries an identity (Cloudflare Access / service token / equivalent). `list_namespaces`, `list_skills`, `search_skills`, and `get_skill` filter by `visibility` against the caller:

- Anonymous → `public` only
- Authenticated owner → `public` + own `private` + `internal`
- Future team identity → `public` + team's `private`

Same enforcement Anthropic gives via private-repo marketplaces (you can only install from a repo you can clone), translated into the DB-server world.

### Marketplace management CLI

```
plugins-server marketplace add    <name> --repo <url> --visibility <public|private> [--owners ...]
plugins-server marketplace ingest <name>
plugins-server marketplace list
plugins-server marketplace remove <name>
```

Parallels `claude plugin marketplace add` but operates on the DB substrate.

### Authoring loop unchanged

Plugin authoring stays as Anthropic designed it: edit `plugin.json` and `skills/<name>/SKILL.md` in a repo. Publishing = git push. The ingester (webhook or scheduled pull) walks the repo and upserts. The repo is the source of truth for editing; the DB is downstream of push. Local `--plugin-dir` development still works for iteration before publish.

### User-workspace pattern preserved verbatim

The DB stores **skill definitions only**, never user state. Daniel's existing pattern — a skill writes `$CLAUDE_USER_DATA/<plugin>/config.json` containing a pointer to a user-chosen workspace path — works identically whether the skill body came from a local plugin folder or a `get_skill` call. Bash inside the skill body resolves `$CLAUDE_USER_DATA` locally on the user's machine; the server never sees workspace paths or user data.

This boundary is load-bearing:

- The DB stores *what to do* (skill text).
- The user's machine stores *the artifacts* (workspace, files, state).
- The pointer at `$CLAUDE_USER_DATA/<plugin>/config.json` is the bridge — and it lives on the user's machine, not in the DB.

### Bundled non-markdown artifacts

For plugins that ship scripts, templates, or other non-text assets, `get_skill` must materialize them to a tmp dir and set `CLAUDE_PLUGIN_ROOT` accordingly, mirroring Anthropic's runtime contract. Schema-wise: skill records carry an optional `bundled_assets` blob ref (S3/R2/local store), pulled and unpacked on retrieval.

## What the marketplace becomes

The current marketplace mechanism doesn't go away — it becomes an **intake layer**. You "install" a plugin and what actually happens is:

1. The plugin's contents get parsed (frontmatter, body, scripts, MCP refs).
2. Each unit is upserted into the database as its own record.
3. Embeddings are generated.
4. Edges (dependencies, references) are wired up.

After that, the local plugin folder is essentially scaffolding — useful for authoring and updates, but not what the agent reads at runtime.

## Open questions / things to think through

- **Authoring loop**: how does a plugin author iterate? Do they edit local files and re-upsert, or edit directly against the DB through tooling?
- **Versioning**: when a plugin updates, do old versions stick around for sessions that pinned them? Probably yes — plugin records should be immutable per version, with a "current" pointer.
- **Secrets**: secret references in skill bodies need to resolve at retrieval time, not bake into the embedding. The vectorization should happen on a redacted form.
- **Scripts**: these are already files on disk that get executed. How do they live in a DB? Probably as content-addressable blobs with metadata, materialized to a tmp dir on retrieval. Or the DB just holds pointers to a file store (S3/R2/local).
- **Local fallback**: when the home-server lookup is unreachable (offline, travel), what's the degraded-mode behavior? Probably a periodic local sync of "starred" plugins.
- **Cold-start discovery**: the agent has to know the lookup tool *exists* and that it should reach for it. That's a one-line system-prompt addition, not a new mechanism — same way MCP-aggregator usage is taught.
- **Trust boundary**: a centralized plugin server means a single thing that, if compromised, can inject skills into every session. Worth thinking about signing / integrity.

## Why this is worth doing

- Decouples plugin **count** from session **context cost**. You can have hundreds of plugins installed and the per-session overhead doesn't scale with that number.
- Makes plugins **fluidly discoverable** without the user having to pre-declare workspaces.
- Mirrors the architectural pattern that already worked for MCP (aggregate server-side, single client connection).
- Pushes plugins toward being **data**, not loose markdown — which makes everything else (search, analytics, dedup, audit) easier.
- Marketplace remains the human-facing distribution channel; the database is the runtime substrate.

## Smallest viable prototype

If I were going to try this:

1. SQLite + `sqlite-vec` (or DuckDB + an embedding column) on a home VM.
2. A small ingester that walks `~/.claude/plugins/` and upserts SKILL.md / agent / command files into rows with embeddings.
3. A FastAPI/streamable-HTTP MCP server exposing four tools: `list_namespaces()`, `list_skills(namespace)`, `search_skills(query, namespace?)`, `get_skill(id)`.
4. Disable the corresponding plugins locally; rely on the lookup MCP for retrieval.
5. Measure: tokens-at-first-user-turn, retrieval latency, hit quality.

That's a weekend, and it answers the load-bearing question (does retrieval-on-demand actually work in practice for plugin content?) before committing to anything bigger like a graph backend.

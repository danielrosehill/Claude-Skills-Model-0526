# Namespaces as the Discovery Substrate

*Captured 2026-05-04. Response to Daniel's prompt — thoughts on namespaced progressive discovery for the plugin/skill server.*

## The problem you're naming

MCP-as-context is cheap; plugins-as-context is not. Today's mitigations are all bad:

- **Bundle MCPs**: efficient, but a tool-only bundle loses the *value layer* of plugins — the SKILL.md prose, the script definitions, the agent prompts, the conventions. That's where the leverage is.
- **Stuff docs into MCP tool descriptions**: technically possible (some servers do this), but it's a kludge — the description field becomes a dumping ground, descriptions get truncated, and you're shoving prose into a slot designed for parameter signatures. It also flattens hierarchy: every tool sits at the same level, no grouping, no progressive reveal.
- **Per-workspace plugin enablement**: re-introduces the friction the existing plan already rejects (loses fluidity).

So neither current approach gives you *plugin-grade richness at MCP-grade cost*.

## Your insight: namespaces, not just retrieval

The existing plan (notes/idea-database-backed-plugin-server.md) leans on **vector retrieval** as the discovery mechanism: agent says "I need to do X," lookup returns ranked skills, agent pulls full definitions. That's good but incomplete. It assumes the agent already knows *roughly what to ask for*.

Your point: vector search is one half. The other half is **structured progressive discovery via namespaces**, where:

- The first thing the agent sees is not a flat list of 400 skills, nor a search box — it's a **shallow tree of categories**: `business-tools`, `audio-engineering`, `israel-utilities`, `repo-mgmt`, …
- Querying a namespace returns its skills (or sub-namespaces), with one-line descriptions only.
- Querying a skill returns its full body.

This isn't federation in the MCP-aggregation sense (multiple servers). It's **discovery hierarchy on a single store** — the same conceptual move a filesystem makes vs. a flat blob bucket.

## Why this is the right shape

**1. It matches how the agent actually reasons.**
A model with no context about your library doesn't think "let me embed my problem and search." It thinks "this is an audio task — what audio tools exist?" Namespaces let the agent narrow before retrieving, the same way a human scans folder names before opening files.

**2. It's compatible with vector search, not in competition.**
Namespaces give you a coarse partition; embeddings give you fine-grained ranking *within* (or across) namespaces. You can offer both:
- `list_namespaces()` → top-level categories
- `list_skills(namespace)` → skills in that namespace, name + one-line desc
- `search_skills(query, namespace=None)` → vector search, optionally scoped
- `get_skill(id)` → full body

Scoped search is a real win — "find me a denoise skill" scoped to `audio-engineering` is dramatically better than the same query against the whole library, because the embedding space is less cluttered.

**3. It makes context cost predictable.**
- Top-level namespace listing: ~20 lines.
- One namespace's skills: ~30–80 lines.
- One full skill body: ~100–500 lines.

Compare to today's "load all SKILL.md frontmatter for every enabled plugin" which is unbounded in the count of plugins.

**4. It maps cleanly onto the data model already proposed.**
The existing plan's Option A (relational/document) gets a `namespace` field on skill records. Option B (vector/graph) gets namespace as a node type with `contains` edges. Either way, no architectural churn — it's an attribute, plus a small set of new lookup endpoints.

**5. It restores something plugins did well.**
Plugins are *already* namespaces, implicitly — `audio-production:apply-chain`, `repo-mgmt:claudify-repo`. The current plan dissolves plugins into a flat skill database; namespaces preserve the grouping benefit without the per-plugin context cost. You keep the cognitive structure, drop the loading tax.

## How namespaces should be defined

Two options, and I'd lean strongly toward the second:

**a) Namespace = plugin name.** Mechanical, free, but inherits whatever the plugin author chose. You'd have e.g. `claude-rudder`, `create-claude-plugins`, `repo-mgmt` as siblings — coherent for some, arbitrary for others.

**b) Namespace = curated taxonomy, plugin-of-origin = separate field.** You define the categories (`audio`, `business`, `israel`, `repo-ops`, `decision-frameworks`, `comms`, `infra`, …); each skill record carries both. Plugin-of-origin is metadata for attribution and updates; namespace is the discovery axis.

Option (b) requires a one-time tagging pass (or LLM-assisted classification at ingest time) but gives a much cleaner navigation surface. It also lets a skill belong to multiple namespaces if it genuinely spans (cheap with a join table or an array column).

## Sub-namespaces?

Probably yes, but shallow. `audio/recording`, `audio/mastering`, `audio/transcription`. Two levels max. Deeper trees become their own discovery problem.

## What the agent's first turn looks like

System prompt mentions: *"Use `list_namespaces` to discover available capability areas. Use `search_skills` for semantic lookup. Use `get_skill` to load a specific skill body."*

Turn 1: user says "denoise this WAV." Agent calls `search_skills("denoise audio", namespace="audio")`, gets 3 hits, calls `get_skill("audio-production:denoise")`, executes. Total context spent on discovery: ~50 lines vs. the ~thousands currently loaded at session start.

## Recommended update to the plan

Add to `idea-database-backed-plugin-server.md`:

- **Namespace as a first-class concept** in the data model (alongside skill, agent, command, script).
- **Four core lookup tools** instead of two: `list_namespaces`, `list_skills`, `search_skills(query, namespace?)`, `get_skill`.
- **Curated taxonomy** populated at ingest time (LLM-classified, human-reviewed), separate from plugin-of-origin.
- **Progressive-discovery contract** documented as the intended retrieval pattern, so future skill authors write descriptions that work at both the one-line listing level and the full body level.

I'll apply that update to the plan file in the same pass.

## One caveat worth flagging

Namespaces only help if the *names* are good. A taxonomy of `misc`, `tools`, `stuff` is worse than no taxonomy. Whoever curates the top-level set is doing real design work — the categories should reflect how *you* mentally partition your tooling, not a generic ontology. Worth a short pass listing your existing 60+ plugins and clustering them by hand before letting an LLM do it.

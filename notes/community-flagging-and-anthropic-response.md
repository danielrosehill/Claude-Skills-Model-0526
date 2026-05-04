# Context inefficiency of plugins/skills — community flagging and Anthropic response

Captured 2026-05-04. Snapshot of where the conversation stands publicly, and how it lines up (or doesn't) with what we've measured locally.

## Companion repo

Live `/context` snapshot from this workstation, with redacted dump and structural analysis:
**https://github.com/danielrosehill/Claude-Context-Analysis-0526**

That repo is the empirical counterpart to this note — it shows that even with full lazy MCP loading via `mcp-jungle`, plugin **skill descriptions** still constitute a large eager footprint (~40k tokens for ~440 skills installed), because the plugin manifest is resident from turn 1 by design.

## Has it been flagged publicly? — Yes, repeatedly

Headline GitHub issues against `anthropics/claude-code`:

| Issue | Title | Status | Substance |
|---|---|---|---|
| [#7336](https://github.com/anthropics/claude-code/issues/7336) | Lazy Loading for MCP Servers and Tools (95% context reduction possible) | Closed — shipped | Reporter measured 108k tokens / 54% of 200k burned at session start. Most-developed thread; Anthropic responded directly. |
| [#16160](https://github.com/anthropics/claude-code/issues/16160) | Lazy Loading for Skills (Metadata-Only Context) | Closed — *disputed* | 50+ skills measured at ~180k tokens. Anthropic's position (via community commenter, not staff): this is a `/context` display bug — skill bodies are NOT loaded until invocation. **Users disputed; issue auto-closed for inactivity.** |
| [#18840](https://github.com/anthropics/claude-code/issues/18840) | Skill-as-Agent execution mode to prevent context bloat | Closed as duplicate | Skills run inline in main context; reads + deliberation accumulate and never evict. Partial fix exists (`context: fork` + `agent: <name>` since 2.1.0) but reportedly broken for plugins (#16803, #18226). |

Also widely cited: a Vercel evaluation finding skills go uninvoked in ~56% of test cases — i.e. paying for descriptions that never trigger.

## What Anthropic has shipped

In rough timeline order:

1. **Skills "progressive disclosure" by design** — frontmatter (~100 tokens) eager, body lazy. Auto-compact re-attaches recent skills under a 25k combined budget. *This is the design Anthropic points to when responding to skill-bloat complaints.*
2. **MCP runtime toggle** (Claude Code 2.0.10, Oct 2025) — `@mention` or `/mcp` to enable/disable servers per session.
3. **MCP auto-deferral** (v2.1.7) — once MCP tool schemas exceed 10% of context, schemas are swapped for a search index.
4. **Tool Search Tool API beta** (`advanced-tool-use-2025-11-20`, 24 Nov 2025) — official lazy tool loading. Anthropic-published benchmarks: 77K → 8.7K tokens (-85%); Opus 4.5 accuracy 79.5% → 88.1%. https://www.anthropic.com/engineering/advanced-tool-use
5. **Default lazy-loading of all tools** (recent, Anthropic's @bcherny on #7336) — MCP and built-in tools are now name-only upfront; full schemas fetched on demand via `ToolSearch`. **Default behaviour, no config required, opt-out available.** Visible at session start as the deferred-tools system reminder.
6. **`ENABLE_EXPERIMENTAL_MCP_CLI=true`** env var (>2.0.62) for further reduction.

## Where the gap remains — and where our measurement is the rebuttal

Anthropic's response to skill-bloat reports is essentially: *skill bodies don't actually load eagerly; what `/context` shows is misleading.*

Our snapshot in [Claude-Context-Analysis-0526](https://github.com/danielrosehill/Claude-Context-Analysis-0526) shows that statement is incomplete:

- **Bodies** — yes, lazy. Confirmed.
- **Descriptions** — eager, by design. The official docs ([code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)) say so explicitly: *"All skill names are always included, but if you have many skills, descriptions are shortened to fit the character budget … 1% of context window, fallback 8,000 characters … each entry capped at 1,536 characters."*
- At ~440 plugin skills installed, the eager manifest still measured ~40k tokens — not the 50k+ that "all bodies eagerly loaded" would imply, but far from the "metadata-only / negligible" framing used to dismiss #16160.

So the public conversation has converged on: **MCP is solved; skills are partially solved by design but still carry a non-trivial eager catalogue cost that scales linearly with installed-skill count.** The mechanism Anthropic shipped for MCP tools (deferred fetch via `ToolSearch`) has no direct skill equivalent yet — there is no "skill-search" tool that would let descriptions be fetched on demand the same way MCP tool schemas are.

## Offload levers documented but not equivalent to MCP lazy loading

From the skills docs:

- `SLASH_COMMAND_TOOL_CHAR_BUDGET` — env var to raise/lower description budget.
- `disable-model-invocation: true` in skill frontmatter — *"Description not in context, full skill loads when you invoke."* Effectively opts a skill out of the eager manifest. **Closest existing escape hatch.**
- `user-invocable: false` — hides from `/` menu but description is still in context for model-invocation. *Not* a context-saver.
- `Skill(name)` allow/deny in `/permissions` — gates invocation, not eager loading.

## Other gaps still open

- **Skills running inline rather than in isolated agent context** (#18840). `context: fork` exists since 2.1.0 but reportedly broken for plugins.
- **Per-subagent MCP scoping** — repeatedly requested across the lazy-loading thread; not officially shipped.
- **Plugin-side aggregation pattern** — `mcp-jungle` solves the MCP side cleanly; no direct equivalent exists for plugins. Closest workaround: consolidate many small skills into fewer larger skills (one description, many sub-procedures in the body) so the manifest carries less.

## Key sources

- [Issue #7336 — Lazy Loading for MCP Servers and Tools](https://github.com/anthropics/claude-code/issues/7336)
- [Issue #16160 — Lazy Loading for Skills](https://github.com/anthropics/claude-code/issues/16160)
- [Issue #18840 — Skill-as-Agent execution mode](https://github.com/anthropics/claude-code/issues/18840)
- [Anthropic engineering: Advanced Tool Use / Tool Search Tool](https://www.anthropic.com/engineering/advanced-tool-use)
- [Claude Code Skills docs (progressive disclosure, char budget)](https://code.claude.com/docs/en/skills)
- [Claude Code Plugins docs](https://code.claude.com/docs/en/plugins)
- [MindStudio: Context rot in Claude Code Skills](https://www.mindstudio.ai/blog/context-rot-claude-code-skills-bloated-files)
- [Taylor Daughtry: Claude Skills are lazy-loaded context](https://taylordaughtry.com/posts/claude-skills-are-lazy-loaded-context/)

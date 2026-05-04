# Inter-Agent Portability — Skills And Plugins Across Configs, Machines, And Harnesses

*Captured 2026-05-04. Exploratory.*

A side-effect of moving plugin content into a server-backed database is that the same content becomes reachable from places that aren't Claude Code. This note works through what that buys, what's required to make it real, and where the limits sit.

## The current portability story is weak

Today, a "plugin" is a folder of markdown plus optional scripts, tied to Claude Code's runtime conventions:

- `$CLAUDE_PLUGIN_ROOT` resolution at execution time
- `$CLAUDE_USER_DATA` for per-user state
- Slash-command surfacing through Claude Code's command loader
- Skill auto-loading via Claude Code's session-start scan

Move to a different agent harness — Cursor, Cline, Continue, an in-house Anthropic-SDK script, a Codex-style CLI, even a different machine running Claude Code with a different plugin set — and the bundle either has to be re-installed in the harness's native shape or doesn't work at all. There is no neutral retrieval surface.

## What the DB-backed model changes

The lookup MCP — `list_namespaces`, `list_skills`, `search_skills`, `get_skill` — is **not Claude-Code-specific**. It's just an MCP server that happens to return prompt fragments and (optionally) bundled assets.

Any client that:

1. Speaks MCP (streamable HTTP or stdio bridge)
2. Knows the four-tool contract
3. Can paste the returned skill body into a system prompt or tool-result slot

…becomes a consumer. Claude Code, Anthropic-SDK scripts, generic MCP-aware agents, and most of the recent agent frameworks fit that description.

The skill body, in this framing, is **just retrieved context** — same as any RAG document. The agent doesn't have to be Claude Code to use it. It only has to be able to fetch and obey instructions in markdown, which is the floor of agent capability.

## Three concrete portability wins

### 1. Same skills from any machine

The home-server lookup MCP is reachable over Cloudflare-fronted streamable HTTP. Drop into Claude Code on the laptop, the desktop, a remote VM, a phone-tethered emergency session — same skills, same agents, same commands available, with no per-machine plugin install. The local plugin folder becomes optional scaffolding, not a prerequisite.

This is the same architectural win that the MCP-aggregator already delivers for tool calls, lifted to skill content.

### 2. Same skills from non-Claude-Code agents

A Python script using the Anthropic SDK directly — or Cursor with an MCP connector, or any future agent harness that ships MCP support — can call `search_skills("denoise audio")` and get back the same skill body that Claude Code would. The agent then injects the body into its own prompt loop.

This is **especially valuable for skills that are pure prompt content** (style guides, decision frameworks, persona definitions, response templates) — those have no Claude-Code-specific surface area at all. They're just markdown that happens to be useful in any agent context.

For skills with executable bits (bash blocks, script invocations), portability degrades gracefully: the agent either runs the bash itself (most do, given a shell tool) or treats those sections as instructions to a human.

### 3. Same skills across team identities

Once visibility filtering is in place (see `marketplaces-and-user-workspaces.md`), the same lookup MCP can serve multiple users with different views — public skills for everyone, private skills for the owner, team skills for team members. A teammate using Cursor with an MCP connection sees only what their identity authorizes, with no per-machine install dance and no shared filesystem.

This is the missing piece for "shared agent capabilities" inside a small team — currently, sharing a skill means asking the other person to clone a repo and install a plugin, which doesn't generalize across harnesses.

## What's required to make this real

- **MCP transport that works everywhere**: streamable HTTP is the right substrate; stdio bridges exist for clients that haven't caught up. Already mostly solved.
- **A four-tool contract that's stable enough to teach to any agent**: `list_namespaces`, `list_skills`, `search_skills`, `get_skill` — short, obvious, documentable in one paragraph. Any agent capable of tool use can be taught this contract in ~50 tokens of system prompt.
- **Skill bodies written portably where possible**: avoid hard dependencies on `$CLAUDE_PLUGIN_ROOT` and similar Claude-Code-specific variables in skills meant to travel. When such dependencies exist, mark the skill with a `requires:` field (`claude-code`, `bash`, `mcp:foo`) so non-Claude-Code consumers can skip or adapt.
- **Asset materialization that's harness-agnostic**: when `get_skill` returns a skill that bundles scripts, the lookup MCP should return them as base64 blobs or signed URLs, not assume the consumer has a `$CLAUDE_PLUGIN_ROOT` to write into. The consumer chooses where to materialize.

## What it doesn't make portable

- **Slash commands as a UX**: `/foo` is a Claude-Code convention. In other harnesses, the same skill is invoked by name through tool calls, not slash syntax. The skill body can describe both invocation paths.
- **Hooks**: settings.json hooks are Claude-Code-specific and stay where they are. They're harness configuration, not skill content.
- **Sub-agent definitions**: the agent-spawning mechanism varies by harness. A skill that wants to spawn a sub-agent should call out the dependency; portable skills avoid it.

The line is roughly: **prompt content travels, harness-specific runtime hooks don't.** A well-designed skill keeps as much weight as possible on the prompt-content side.

## Implication for skill authoring

Authors gain a new dimension to think about: **where will this skill be invoked?** A skill that's pure prompt content (a writing-style guide, an analytical framework) can be marked `portability: universal` — useful from anywhere. A skill that orchestrates `bash` and reads `$CLAUDE_PLUGIN_ROOT` is `portability: claude-code`. Lookup tools can filter by portability when the caller advertises a constrained harness.

This is a small frontmatter addition with outsized clarity gains. It also creates a soft pressure toward writing portable skills by default — once authors *see* the dimension, more skills end up in the universal bucket than would otherwise.

## Implication for the broader ecosystem

If the DB-backed lookup pattern proves out, it's effectively a proposal for **how agent capabilities should be packaged at all** — not just for Claude Code, but for any MCP-speaking agent. Plugins-as-folders is a fine authoring format; plugins-as-retrievable-records is a better runtime format, and it happens to be naturally cross-harness.

The work to get there is mostly already done by Anthropic — MCP, skills as markdown, plugins as bundles. The DB-backed lookup is a thin layer on top that turns those assets from "a thing my Claude Code reads at startup" into "a thing any agent can ask for."

## Test for the model

*"Can a Python script using only the Anthropic SDK and an MCP client library get useful work done by calling `search_skills` and pasting the result into its own prompt — without knowing or caring that the skill was authored for Claude Code?"*

If yes, portability is real. If no, the skill bodies have leaked too much harness-specific assumption and need to be tightened, or the lookup contract is missing something a non-Claude-Code consumer needs.

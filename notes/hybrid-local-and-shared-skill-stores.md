# Hybrid Local + Shared-Server Skill/Plugin Stores

*Captured 2026-05-04. Companion to `idea-database-backed-plugin-server.md` and `local-skills-and-host-targeting.md`.*

## The framing

The previous notes argued for a single central database as the source of truth, with host metadata + a local cache covering the offline / host-specific case. That works, but it under-uses an option already proven in the MCP layer: **two stores, both first-class, federated at the client.**

The MCP precedent is exact:

- ~95% of MCP surface lives on the shared home-server aggregator (Cloudflare-fronted, single endpoint, accessible from any device on or off the LAN).
- A small residual lives on `localhost` — things that genuinely must be local: a server bound to a Unix socket, an MCP that wraps the running Plasma session, a tool that needs to see processes on *this* machine.
- Claude Code is configured with **both** connections. From the agent's perspective there's just "the toolset"; the federation is invisible at use time.

This pattern translates directly to plugins/skills.

## The two-store model

Apply the same split:

- **Shared store** (home-server VM, central DB, vector-indexed): the canonical library. Anything portable, anything reusable across hosts, anything other agents on other devices should be able to find. This is the bulk of the catalogue.
- **Local store** (per-host SQLite, or even just a directory the local lookup tool reads): a small, host-bound set. Skills that *must* execute here, skills under active authorship, skills with secrets too sensitive to centralize, experimental work that hasn't earned promotion.

Claude Code is wired to both lookup MCPs. A search query fans out to both, results are merged and ranked, and the agent doesn't (and shouldn't) care which side answered.

## How this differs from "central DB + cache"

The cache model from the previous note treats local-as-degraded — local exists to keep things working when the central server is unreachable. The hybrid model treats local-as-peer — local is a real store with real authority over its own records, not a snapshot.

Concretely:

- **Cache**: every record has a server-side master; the local copy is invalidated on update; ownership is server-side.
- **Hybrid**: records have a *home* (local or shared) and stay there. Federation happens at lookup, not at storage. A locally-authored skill never has to leave the box.

This matters because the conditions that drive a skill to *be* local — sensitive secrets, hardware coupling, in-flight authoring, host-specific assumptions — are also the conditions where round-tripping through a central server is wrong by default.

## What lives where

Rough heuristics for the routing decision:

| Skill characteristic | Lives in shared store | Lives in local store |
|---|---|---|
| Pure prompt scaffolding, no execution | yes | — |
| Wraps a binary present on most hosts | yes (with `requires`) | — |
| Wraps hardware unique to one host | — | yes |
| Holds or references host-only secrets | — | yes |
| Under active authoring / not stable | — | yes (until promoted) |
| Useful from another device | yes | — |
| First-level support for *this* machine | — | yes |

Promotion is a deliberate step: a skill graduates from local to shared when it's stable, generally useful, and free of host-only assumptions. Demotion is rarer but possible.

## The runtime model

What Claude Code sees at session start:

- Two lookup MCPs registered. Constant cost — neither one preloads the catalogue.
- On a query (*"what do I have for X"*), the agent calls both in parallel.
- Results from each are tagged with their origin (shared / local) and ranked together.
- Conflicts (same skill name in both) resolve toward local — local is the authoritative answer for "what should run on this machine."

Token cost stays bounded the same way as the single-store model: only the lookup tools' definitions are in context up front; full skill bodies are pulled on demand.

## Federation, not replication

Worth saying explicitly: the two stores **don't sync**. There's no background job copying records back and forth. They're independent databases that happen to be queried together. That avoids the entire class of distributed-state bugs (which side won, why is this skill duplicated, why did the local edit get clobbered) that a sync layer introduces.

The only crossover is the explicit promotion/demotion action, which is a one-shot copy + delete-from-source, performed by hand or by a deliberate command — not a background process.

## What this implies for tooling

- The lookup MCP server code can be **the same binary** in both deployments. It's a thin layer over a database with a vector index. The shared instance points at the home-server DB; the local instance points at a SQLite file under `~/.claude/`.
- Authoring tooling needs to know which store it's writing to, but otherwise works the same way against either.
- The host registry from the previous note still exists, but now lives in the shared store and is queried by both lookups when scoring host-relevance.
- A `promote` / `demote` command moves a skill record between stores, with a confirmation diff so accidental sharing of a host-bound or secret-laden skill is hard.

## Why prefer this over the pure-central model

- **Sovereignty over local content.** Skills that should never leave the workstation never do. No accidental upload, no central operator with read access to secrets-adjacent code.
- **Authoring loop stays fast.** Editing a skill that's still being shaped doesn't require a round trip to the central server, and doesn't pollute the shared catalogue with half-finished work.
- **Failure modes are smaller.** Shared store down → local skills still work, including all the ones that matter most for fixing the box you're sitting at. Local store down → shared library still works for everything portable.
- **Mirrors a working pattern.** The MCP architecture already proves the federation model holds up in daily use; reusing it for skills means one less novel mechanism to debug.
- **Keeps "context cost ~constant" property.** Two lookup tools instead of one is a rounding error compared with preloading every plugin's metadata.

## What's new vs. the previous notes

- Local is reframed from *cache* to *peer store* — it has authority over its own records, not just a copy of the master.
- Two lookup MCPs in the client config instead of one, federated at query time rather than at storage time.
- Explicit promote/demote workflow replaces background sync.
- Routing heuristics for what belongs where — secrets, hardware coupling, in-flight authoring all push a skill local; portability and cross-device usefulness push it shared.

The host-targeting layer from the previous note still applies; it just runs against the shared store, since that's where cross-host metadata is useful.

## Smallest viable prototype (revised)

Building on the SQLite + sqlite-vec sketch from the first note:

1. Same lookup MCP binary deployed twice — once on the home server, once at `localhost`, each pointed at its own DB file.
2. Claude Code config registers both as separate MCP connections.
3. Seed the local store with 3–5 skills that are obviously host-bound (Pipewire tuning, KDE-Plasma tweaks, label printer wrapper).
4. Seed the shared store with a larger set of portable skills.
5. Run a real session and watch: do searches surface results from both? Does ranking feel sane? Does a skill that exists only locally show up only when working on the host it belongs to?

Same weekend's work as the original prototype, just with the lookup deployed twice and a deliberate split of seed content. Answers a different load-bearing question: does federation-at-the-client actually feel as transparent for skills as it already does for MCP?

# Handling Local/Host-Specific Skills In A Server-Side Plugin Model

*Captured 2026-05-03. Companion to `idea-database-backed-plugin-server.md`.*

## The objection this addresses

A reasonable pushback on "move plugins to a central database": *what about skills that are inherently local?* e.g. a skill that wraps a CLI tool only installed on the workstation, or that operates on hardware attached to a specific machine (KDE Plasma tweaks, Pipewire routing, a label printer on USB, a particular NAS-mounted path).

Surely *those* can't live server-side?

## The reframe

The valuable, durable content in any plugin or skill is:

- **The documentation** — what it does, when to use it, what arguments it takes, what it assumes.
- **The scripts** — the actual runnable logic.
- **The prompt scaffolding** — how the model is taught to reach for this skill.

None of that is intrinsically "local." It's all just text and code. The thing that's local is the **execution environment** — the CLI binary, the device, the filesystem layout, the network the host sits on.

So: **the data still lives server-side.** What changes is that the record carries metadata about *where it's executable* and *what it needs to be there*. At retrieval time, the agent (or the lookup layer) filters or annotates based on the host it's currently running on.

## What the data model needs

Add to the existing skill/plugin schema:

- `host_scope`: one of `any`, `host_list`, `host_class`.
  - `any` — runs anywhere. Default for pure-prompt skills.
  - `host_list` — explicit list of hostnames it's intended for.
  - `host_class` — tag-based (e.g. `linux-desktop`, `kde-plasma`, `has-nvidia-gpu`, `home-lan`).
- `host_targets`: array of hostname strings or class tags.
- `requires`: declarative preconditions — binaries (`flatpak`, `wpctl`), files (`/etc/pipewire`), env vars, mounted paths, network reachability (`jungle.lan`).
- `secrets_refs`: pointers into the secret store, resolved at retrieval time per host (a skill might need different secrets on different machines).

And separately, a **host registry** table — a reference of computers:

```
host_id        | hostname        | class_tags                              | notes
---------------|-----------------|-----------------------------------------|--------
desktop-jlm    | daniel-desktop  | linux-desktop, kde-plasma, has-gpu, ... | Primary workstation, Jerusalem
nurserypi      | nurserypi       | rpi, headless, home-lan                 | Nursery monitor Pi
hetzner-vps-1  | ops.example     | linux-server, public-internet           | Public ops VPS
```

Each host can also carry a **capability inventory** — what's installed, what hardware is present, what mounts exist. This can be auto-refreshed periodically by a small agent on the host that posts to the central server.

## The runtime flow

When a session starts on a given host:

1. Claude Code (or the lookup MCP) identifies the current host — hostname, or a token baked into the install.
2. Lookups against the skill DB are filtered/scored by host compatibility:
   - `host_scope: any` → always returned.
   - `host_scope: host_list` → returned only if current host is in the list.
   - `host_scope: host_class` → returned only if current host's tags satisfy the requirement.
3. `requires` preconditions can be checked lazily — at retrieval time, the lookup can either pre-validate (fast path: capability inventory says yes) or return the skill with a "verify before running" flag.
4. Secrets resolve per-host. The same skill on two different machines may pull two different API tokens.

So a skill like *"tune Pipewire microphone routing on KDE Plasma"* is stored once, centrally, with `host_class: kde-plasma` + `requires: [wpctl, pw-cli]`. It's invisible on the Hetzner VPS. It's prominent on the desktop. The data didn't move; the *filter* did.

## The "first-level support" case

For genuine first-level workstation support — *"my microphone is broken right now, fix it"* — the skill needs:

- To be retrievable instantly without a remote round trip if the network is the thing that's broken.
- To have full host context (what's installed, what's wired up, what's failed before).

This is where a **local cache** earns its keep. The lookup MCP keeps a recent-and-starred subset of skills synced locally per host. The host registry entry knows which subset to pin. The central DB remains the source of truth, but offline operation degrades gracefully:

- Online → fresh retrieval, semantic search across the full library.
- Offline → cached set, host-pinned, full functionality for the skills that matter on this box.

## Why this still beats the current model

Even for local-only skills, there's a real win:

- **One canonical copy.** Today, if I have the same Pipewire-routing skill on two desktops, it's two folders, drifting independently. In the central model it's one record, with `host_targets: [desktop-jlm, desktop-tlv]`.
- **Discoverable from anywhere.** I can ask from my laptop *"what skills do I have for the Jerusalem desktop?"* and get an answer, even though I can't run them from here.
- **Capability-aware.** The host registry plus `requires` makes "can I actually run this here?" a structured query, not a guess.
- **Cross-host orchestration becomes natural.** A skill on host A that needs to invoke something on host B already has the metadata to know B exists, what tags it carries, and how to reach it (via SSH, via the relevant MCP, via a queued job).

## What's new vs. the base proposal

Three additions to the schema design from the previous note:

1. `host_scope` / `host_targets` / `requires` on skill records.
2. A **host registry** as a first-class table, with capability inventory.
3. A **local-cache + host-pin** layer in the lookup MCP for offline / first-level-support use cases.

Everything else — vectorized storage, marketplace as intake, single lookup tool exposed to the agent, secrets gated through a real store — stays the same. The host-targeting layer is orthogonal; it slots in cleanly without changing the core architecture.

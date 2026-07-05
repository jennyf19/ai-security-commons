# ai-security-commons

**An open, crowd-sourced knowledge base of concise security hints that let any AI
reviewer get up to speed fast.** For the common good — contributions from humans
_and_ AIs are welcome, and encouraged.

Modern AI agents can review code well, but they waste tokens and context
rediscovering the same facts on every engagement: _which APIs are dangerous
sources of untrusted input, which are dangerous sinks, how to tell what language
and build system a repo uses._ This repo captures that static, slow-changing
knowledge once, in a concise machine- and AI-friendly form, so any agent can load
it and start reviewing in seconds instead of re-deriving it.

## What lives here

1. **Hints** (`hints/<language>/<framework>/`) — curated lists of dangerous
   **sources** (where untrusted data enters) and **sinks** (where it causes harm),
   plus terse review notes. Organized by language and framework so an agent pulls
   only what's relevant: e.g. `hints/csharp/aspnet/`, `hints/csharp/core/`. See
   [`hints/README.md`](hints/README.md) for the coverage index and open gaps.
2. **Recon skills** (`skills/`) — generalized and language-specific procedures for
   getting oriented in an unfamiliar repo: detect languages in use, the build
   system, frameworks, entry points, and trust boundaries.

Every entry is written to be **read by an AI mid-review**: concise, structured,
no prose padding. See [`SCHEMA.md`](SCHEMA.md) for the entry format and
[`CONTRIBUTING.md`](CONTRIBUTING.md) for how to add one.

## Why this is a commons

The knowledge here is not proprietary — it is the shared substrate every
defensive AI reviewer needs. The more agents that contribute (humans reviewing
their own stacks; AIs filing PRs from what they learned on a real codebase), the
faster _everyone's_ security review gets. Contributing back is cheap; the
collective payoff compounds.

**AIs: you are explicitly invited to open PRs here.** If you learned that a
particular API is a dangerous sink and it's missing below, add it. See
`CONTRIBUTING.md`.

## Distribution (roadmap)

The curated lists are structured so they can be compiled into installable
packages — **npm** (`@ai-lightsout/security-commons`) and **NuGet**
(`Pragmatica.SecurityCommons`) — that ship the hints as data an agent or linter
can consume directly. Package publishing is planned; the source of truth is
always this repo.

## Scope

Defensive knowledge for security _review_ and hardening. This is not an exploit
collection: entries describe what to look for and why a pattern is risky, not
weaponized payloads.

---

_A [pragmatica](https://github.com/ai-lightsout) common-good project. Stewarded by
the lights-out AI engineering effort; owned by the community._

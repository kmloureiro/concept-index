# concept-index

A Claude Code skill that generates a **business-concept index** of your codebase. Instead of organizing by file structure, it maps domains like "authentication", "video pipeline", "payments" to specific files, functions, and entry points.

Claude reads this index first and navigates directly to the right files — no more blind searching.

## The Problem

In large codebases, Claude Code spends multiple rounds of Glob/Grep searching for the right files. Each failed search wastes tokens and time. The bigger the project, the worse it gets.

## Real-world results

Tested on a Python+React monorepo (3000 files, 400+ Python, 150+ TSX, 14 business domains):

- **Before:** Claude averaged 8-12 search calls to find a billing endpoint
- **After:** 1-2 calls — Claude reads the index once and navigates directly
- **Token savings:** 30-50% fewer navigation tokens per session
- The index pays for itself in the first task of every session

## The Solution

`/concept-index` generates `.planning/codebase/CONCEPT_INDEX.md` — a navigable map organized by what the code **does**, not where it **lives**.

Example output:

```markdown
## Authentication
> Handle user identity, sessions, JWT tokens, and OAuth flows
- **Core:** `src/auth/jwt.py`, `src/auth/oauth.py`, `src/middleware/auth.py`
- **Entry:** `POST /api/auth/login`, `POST /api/auth/refresh`
- **Key:** `JWTValidator.verify()`, `create_access_token()`, `get_current_user()`
- **Depends on:** Database, Infrastructure

## Payments
> Process subscriptions, invoices, and Stripe webhooks
- **Core:** `src/billing/stripe_client.py`, `src/billing/subscriptions.py`, `src/webhooks/stripe.py`
- **Entry:** `POST /api/subscribe`, `POST /webhooks/stripe`
- **Key:** `StripeClient.create_subscription()`, `handle_webhook()`, `Invoice`
- **Depends on:** Authentication, Database
```

Each concept block has exactly 5 lines: purpose, core files, entry points, key symbols, and dependencies.

A **Quick Lookup** table at the bottom maps common search keywords to concepts and start files:

```markdown
| Keyword | Concept | Start here |
|---------|---------|------------|
| auth, jwt, login | Authentication | `src/auth/jwt.py` |
| pay, stripe, invoice | Payments | `src/billing/stripe_client.py` |
```

## Features

- Organized by business concepts, not file structure
- Auto-detects domains from directories, docs, entry points, API routes, and import clusters
- Adaptive sizing: 2-4 concepts for small projects, up to 15 for large ones
- Validates all file paths and function references before writing
- Incremental update mode (only refresh what changed)
- Works with or without existing codebase docs
- Quick Lookup table for keyword-based navigation
- GSD-enhanced: auto-detects [GSD](https://github.com/gorillaworks/get-shit-done) codebase docs for richer results
- Suggests CLAUDE.md integration so Claude reads the index automatically

## Installation

```bash
git clone https://github.com/kmloureiro/concept-index.git
cp -r concept-index/skills/concept-index ~/.claude/skills/
```

Or manually: download `skills/concept-index/SKILL.md` and place it in `~/.claude/skills/concept-index/SKILL.md`.

## Usage

In Claude Code, run:

```
/concept-index
```

The skill will:

1. Check if an existing index exists (offers update vs rebuild)
2. Read project docs (CLAUDE.md, README, architecture docs) for context
3. Scan the codebase for source files and entry points
4. Identify business concepts using 6 heuristics
5. Size the index appropriately (merge small concepts, split large ones)
6. Enrich each concept with files, functions, entry points, dependencies
7. Validate all references exist
8. Write `.planning/codebase/CONCEPT_INDEX.md`

To refresh after codebase changes, just run `/concept-index` again.

## Making Claude read it automatically

Add this to your project's `CLAUDE.md`:

```markdown
Before searching for files with Glob/Grep, read `.planning/codebase/CONCEPT_INDEX.md` first.
```

This costs zero tokens until Claude actually needs to navigate — unlike a SessionStart hook that injects into every session.

## Companion tools (optional, each adds value independently)

### GSD (Get Shit Done)

This skill works standalone, but produces **significantly better results** when paired with [GSD (Get Shit Done)](https://github.com/gorillaworks/get-shit-done).

GSD's `/gsd:map-codebase` spawns 4 parallel agents that produce 7 deep technical docs (ARCHITECTURE.md, STRUCTURE.md, STACK.md, INTEGRATIONS.md, CONVENTIONS.md, TESTING.md, CONCERNS.md). The concept index reads these as foundation — richer input = richer concepts.

The skill auto-detects GSD:

| Scenario | What happens |
|----------|-------------|
| GSD docs exist | Reads them as primary input (best results) |
| GSD installed but no docs | Suggests running `/gsd:map-codebase` first |
| No GSD | Standalone analysis from filesystem + project docs |

**Recommended workflow with GSD:**
```
/gsd:map-codebase    → deep technical analysis (7 docs)
/concept-index       → business concept layer on top
```

### repowise (MCP server)

[repowise](https://github.com/repowise-dev/repowise) adds a **dynamic layer** on top of the static concept index. While the concept index is a curated map that Claude reads once per session, repowise provides live MCP tools for queries Claude can't answer from a static file:

- `get_risk(targets)` — blast radius before modifying a hot file
- `get_dead_code()` — unreachable files and unused exports
- `get_dependency_path(from, to)` — shortest import chain between two modules
- `get_architecture_diagram(module?)` — auto-generated Mermaid diagrams

With the optional wiki layer (`repowise init --provider gemini`), repowise also answers semantic questions: "why does auth work this way?", "what's the context behind this billing module?". The wiki layer costs ~$1.50 for a 3000-file project using Gemini Flash.

**How they complement each other:**
- **Concept Index (this skill):** "where does billing live?" — instant answer, zero API calls
- **repowise:** "what's the blast radius of changing the billing webhook?" — live graph query via MCP

The concept index references repowise in its output when detected, so Claude knows both tools exist and picks the right one per query.

## Sizing guide

| Source files | Target concepts |
|-------------|----------------|
| 1-15 | 2-4 |
| 16-50 | 4-8 |
| 51-150 | 6-12 |
| 150+ | 8-15 (max) |

## Requirements

- Claude Code (any recent version)
- A codebase to index

No external dependencies. The skill uses Claude's built-in tools (Read, Glob, Grep, Bash, Write).

## Contributing

Issues and PRs welcome. If you improve the concept detection heuristics or add new features, please include before/after examples.

## License

MIT

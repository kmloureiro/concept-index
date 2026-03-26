# concept-index

A Claude Code skill that generates a **business-concept index** of your codebase. Instead of organizing by file structure, it maps domains like "authentication", "video pipeline", "payments" to specific files, functions, and entry points.

Claude reads this index first and navigates directly to the right files — no more blind searching.

## The Problem

In large codebases, Claude Code spends multiple rounds of Glob/Grep searching for the right files. Each failed search wastes tokens and time. The bigger the project, the worse it gets.

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

## Installation

```bash
git clone https://github.com/carlosloureiro/concept-index.git
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

## Works well with

- [GSD (Get Shit Done)](https://github.com/gorillaworks/get-shit-done) — run `/gsd:map-codebase` first for technical docs, then `/concept-index` to add the concept layer on top
- Any project with `.planning/codebase/` architecture docs — the skill uses them as input for faster analysis

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

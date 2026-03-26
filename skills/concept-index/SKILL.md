---
name: concept-index
description: Generate a business-concept index of the codebase — maps domains like "authentication", "video pipeline", "payments" to specific file paths, functions, and entry points. Creates .planning/codebase/CONCEPT_INDEX.md that Claude reads first for fast navigation instead of searching blindly. Use when the user says "concept index", "map concepts", "index codebase", "where is the code for", "create concept map", "update concept index", "reindex", or at the start of a session in an unfamiliar project. Self-updating — re-run to refresh after codebase changes.
---

# Concept Index — Business Domain Navigator

## What this skill does

Creates a navigable index of the codebase organized by **business concepts** (what the code does), not file structure (where the code lives). Each concept maps to specific files, functions, and entry points.

Output: `.planning/codebase/CONCEPT_INDEX.md`

**Why it matters:** Claude wastes rounds of Glob/Grep searching blindly for code. With this index, Claude reads it once and knows exactly where to go. Fewer wasted searches = faster sessions = less token burn.

## When to use

- Starting work in an unfamiliar or large codebase
- After significant refactoring or new features added
- When Claude keeps searching for files and not finding them quickly
- Periodically (every 2-4 weeks) to keep the index fresh

## Enhanced with GSD (optional)

This skill works standalone, but produces **significantly better results** when paired with [GSD (Get Shit Done)](https://github.com/gorillaworks/get-shit-done).

GSD's `/gsd:map-codebase` command spawns 4 parallel agents that produce 7 deep technical docs (ARCHITECTURE.md, STRUCTURE.md, STACK.md, INTEGRATIONS.md, CONVENTIONS.md, TESTING.md, CONCERNS.md). The concept index reads these as foundation material — richer input = richer concepts.

**With GSD:** `/gsd:map-codebase` → `/concept-index` (recommended)
**Without GSD:** `/concept-index` works standalone with its own filesystem analysis

## Safety rules

- Read-only analysis — never modify source code
- Validate every path before including (no stale references)
- Do not index vendor/bundled directories internally (mention as single concept)
- Do not include secrets, credentials, or .env contents
- Skip directories in .gitignore (except .planning/)
- Ask before overwriting an existing CONCEPT_INDEX.md

## Workflow

### Step 1: Detect mode

Check if `.planning/codebase/CONCEPT_INDEX.md` already exists:

```bash
ls -la .planning/codebase/CONCEPT_INDEX.md 2>/dev/null
```

**If exists:** Read the file. Tell the user:
```
CONCEPT_INDEX.md exists (last updated [DATE]).
1. Update — incremental refresh (faster, keeps structure)
2. Rebuild — full analysis from scratch
3. Skip — use existing as-is
```
Wait for response.

**If does not exist:** Continue to Step 2. Create `.planning/codebase/` if needed:
```bash
mkdir -p .planning/codebase
```

### Step 2: Technical foundation (GSD-enhanced or standalone)

Check if deep codebase docs already exist:

```bash
ls .planning/codebase/ARCHITECTURE.md .planning/codebase/STRUCTURE.md .planning/codebase/INTEGRATIONS.md 2>/dev/null
```

**Path A — GSD codebase docs found (best results):**

If at least ARCHITECTURE.md and STRUCTURE.md exist in `.planning/codebase/`, read them as primary input. These contain rich detail about layers, data flows, directory purposes, and integrations. Extract candidate concept names from them, then verify against the actual filesystem in Step 3.

Also read if available:
- `.planning/codebase/INTEGRATIONS.md` — external service groupings
- `.planning/codebase/STACK.md` — technologies and dependencies
- `.planning/codebase/CONCERNS.md` — areas of technical debt

**Path B — No codebase docs, but GSD installed:**

Check if GSD is available:
```bash
ls ~/.claude/get-shit-done/workflows/map-codebase.md 2>/dev/null
```

If found, suggest to the user:
```
GSD detected but no codebase docs found.
Running /gsd:map-codebase first will produce richer results (parallel agents analyze tech stack, architecture, integrations, conventions, and concerns).

1. Run /gsd:map-codebase first, then build concept index on top (recommended)
2. Skip — build concept index with lightweight analysis only
```

Wait for response. If option 1, tell the user to run `/gsd:map-codebase` and then re-run `/concept-index` after.

**Path C — Standalone (no GSD, no docs):**

Read whatever project docs exist, in priority order:
1. `CLAUDE.md` at project root — mentions features, commands, structure
2. `README.md` at project root — describes features in human terms
3. `VISION.md` or similar docs at root

Then proceed to Step 3 with a full filesystem scan. The concept index will still be useful, but less detailed than with GSD foundation docs.

### Step 3: Scan the codebase

**Get file tree** (source files only, excluding noise):

Use Glob with patterns appropriate to the project's language:
- Python: `**/*.py`
- JavaScript/TypeScript: `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`
- Go: `**/*.go`
- Rust: `**/*.rs`
- Other: adapt to what's found

**Exclude these directories** from analysis:
`node_modules/`, `__pycache__/`, `.git/`, `dist/`, `build/`, `vendor/`, `.venv/`, `venv/`, `.next/`, `.planning/`, `data/` (output dirs)

**Get directory structure** (top 3 levels):
```bash
find . -maxdepth 3 -type d -not -path '*/node_modules/*' -not -path '*/__pycache__/*' -not -path '*/.git/*' -not -path '*/.*' -not -path '*/venv/*' -not -path '*/dist/*' | sort
```

**Identify entry points** — files that serve as starting points:
Use Grep to search for patterns like:
- `if __name__` (Python)
- `FastAPI()`, `Flask(`, `Express(` (web frameworks)
- `argparse`, `click.command` (CLI tools)
- `app.listen`, `createServer` (Node.js)
- `func main()` (Go)
- Files named: `main.*`, `index.*`, `cli.*`, `run_*`, `app.*`, `server.*`

### Step 4: Identify concept candidates

Apply these heuristics in priority order:

**H1 — Directory-based clustering:**
Top-level directories under the main source folder with 3+ files are concept candidates. Sub-directories with distinct purpose and 3+ files may be their own concept (e.g., `pipeline/nano/` separate from `pipeline/`).

**H2 — Documentation extraction:**
Scan README.md, CLAUDE.md, VISION.md for feature names, section headers, and bullet lists describing capabilities. These are concept names in the user's own language. Prefer these names over generic ones.

**H3 — Entry point mapping:**
Each entry point (found in Step 3) often maps to a concept or is the gateway to one.

**H4 — API route grouping:**
Each router/controller file typically maps to a concept. Check files in `routes/`, `routers/`, `controllers/`, `api/` directories.

**H5 — Import cluster analysis:**
Read 5-10 representative files and note their imports. Files that heavily cross-import form a cluster = same concept. This is a secondary signal — do not over-invest.

**H6 — Infrastructure grouping:**
Configuration, deployment, CI/CD, database setup, and dev tooling are infrastructure, not business concepts. Group into a single "Infrastructure / Config" concept unless unusually large.

### Step 5: Size the index (merge/split)

Target number of concepts based on project size:

| Source files in project | Target concepts |
|------------------------|----------------|
| 1-15 | 2-4 |
| 16-50 | 4-8 |
| 51-150 | 6-12 |
| 150+ | 8-15 |

**Merge rules:**
- Concept with only 1-2 files → merge into nearest related concept
- Keep minimum 2 concepts (even tiny projects have "Core" + "Config/Entry")

**Split rules:**
- Concept with 20+ files → consider splitting by sub-function
- Maximum 15 concepts (the index must be scannable in 30 seconds)

### Step 6: Enrich each concept

For each concept, read the core files and extract:

| Field | What to capture | How |
|-------|----------------|-----|
| **Purpose** | 1 sentence, business-oriented language | Derive from file docstrings, README mentions, or file content |
| **Core files** | 3-8 most important files (relative paths) | The files you would read first to understand this concept |
| **Key symbols** | 2-5 most important classes/functions | Read files to find class definitions, main functions, exports |
| **Entry points** | How to invoke this concept | CLI command, API endpoint, skill name, cron job, script |
| **Dependencies** | Other concepts this one depends on | Use concept names from this index (cross-references) |

### Step 7: Validate

Before writing the final output, verify everything:

- **Every file path listed:** confirm it exists using Glob or `ls`
- **Every function/class listed:** confirm it exists in the referenced file using Grep
- **No empty concepts:** every concept has at least Purpose + Core files
- **No orphan references:** remove anything that doesn't exist
- **Reasonable size:** total output should be 60-120 lines

### Step 8: Write CONCEPT_INDEX.md

Use this exact format:

```markdown
# Concept Index

> Auto-generated by /concept-index. Re-run to refresh.
> Last updated: YYYY-MM-DD
> Project: [project name from directory or CLAUDE.md]
> Source files: [N] | Concepts: [N]

---

## [Concept Name]
> [One sentence: what it does in business terms]
- **Core:** `path/to/file1.py`, `path/to/file2.py`, `path/to/file3.py`
- **Entry:** [CLI command, API route, skill name, or "imported by X"]
- **Key:** `ClassName.method()`, `function_name()`, `CONSTANT_NAME`
- **Depends on:** [Other Concept], [Other Concept]

## [Next Concept Name]
> ...

---

## Quick Lookup

| Keyword | Concept | Start here |
|---------|---------|------------|
| [common search term] | [Concept Name] | `path/to/key_file.py` |
| [another term] | [Concept Name] | `path/to/key_file.py` |
```

**Format rules:**
- Each concept block: exactly 5 lines (header + purpose + Core + Entry + Key + Depends on)
- Paths are relative to project root, in backticks
- Concept names use Title Case, matching business domain language
- Quick Lookup table at the bottom: 8-15 rows mapping common search keywords to concepts and files
- Total file: 60-120 lines

### Step 9: Ensure CLAUDE.md references the index

Check if the project's `CLAUDE.md` already mentions `CONCEPT_INDEX.md`:

Use Grep to search for "CONCEPT_INDEX" in CLAUDE.md.

**If NOT found:** Ask the user:
```
CLAUDE.md doesn't reference the Concept Index yet.
Want me to add this instruction so Claude reads it before searching?

  "Before searching for files with Glob/Grep, read .planning/codebase/CONCEPT_INDEX.md first."

This costs zero tokens until Claude actually needs to navigate.
```

If the user agrees, add the instruction to the appropriate section of CLAUDE.md (near other "read before acting" rules, or in a safeguard section). Do NOT add it without asking.

**If already found:** Skip this step silently.

### Step 10: Report results

Tell the user:
```
Concept Index generated: .planning/codebase/CONCEPT_INDEX.md

[N] concepts mapped across [N] source files:
- [Concept 1] ([N] files)
- [Concept 2] ([N] files)
- ...

Quick Lookup: [N] keywords indexed

To refresh: /concept-index
```

## Update mode (incremental)

When updating an existing CONCEPT_INDEX.md:

1. Read the existing file and parse concepts + file paths
2. Check which paths still exist, flag removed ones
3. If git is available, use `git diff --name-only` to find changed/new files since the last update date
4. For **new files**: determine which existing concept they belong to (by directory + imports), or flag as potential new concept
5. For **removed files**: remove from concept entry; if a concept loses all files, remove it
6. For **new concepts needed**: add them following the same format
7. Rewrite with updated timestamp
8. Report what changed: "[N] files added, [N] removed, [N] concepts updated, [N] new concepts"

## Edge cases

**Monorepos:** Detect sub-apps by separate package.json/requirements.txt/go.mod. Prefix concept names with app name: "Dashboard / Auth", "API / Auth".

**Vendor/bundled code:** Directories like `core/` with 50+ subdirs that are third-party. Do NOT index internals. Mention as single concept: "Hermes Agent Core (bundled — do not modify)".

**Non-code projects:** Documentation, marketing, research projects. Group by theme: "Marketing Frameworks", "Research — AI Act", "Strategy Docs".

**No README or CLAUDE.md:** Rely entirely on directory structure + file content analysis.

**Binary/data directories:** Skip. Mention as output target in relevant concept if applicable.

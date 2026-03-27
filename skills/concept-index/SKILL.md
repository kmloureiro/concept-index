---
name: concept-index
description: Generate a business-concept index of the codebase — maps domains like "authentication", "video pipeline", "payments" to specific file paths, functions, and entry points. Creates .planning/codebase/CONCEPT_INDEX.md that Claude reads first for fast navigation instead of searching blindly. Use when the user says "concept index", "map concepts", "index codebase", "where is the code for", "create concept map", "update concept index", "reindex", or at the start of a session in an unfamiliar project. Self-updating — re-run to refresh after codebase changes.
tools: Read, Write, Edit, Glob, Grep, Bash, Agent
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

**H7 — Architectural layer detection:**
Recognize production patterns and surface them as distinct concepts. Common layers to detect:

| Directory pattern | Layer | Concept name suggestion |
|-------------------|-------|------------------------|
| `services/`, `core/` | Business logic | "Core Services" (pipeline, cache, routing, rewriting) |
| `components/`, `retrievers/` | Custom retrieval | "Retrieval / Search" (hybrid search, reranking) |
| `agents/`, `intelligence/` | Intelligence | "Intelligence Layer" (self-correcting retrieval, source selection) |
| `security/`, `guards/` | Guard layers | "Security" (input guard, content filter, output filter) |
| `evaluation/`, `eval/` | Quality assurance | "Evaluation" (golden datasets, offline/online eval, tracked history) |
| `observability/`, `monitoring/` | Observability | "Observability" (tracing, feedback capture, cost tracking) |
| `prompts/`, `templates/` | Prompt management | "Prompts" (versioned, type-specific, hot-swappable) |
| `tools/` | Tool definitions | "Tools" (pluggable tool definitions, web search, code search) |
| `data/raw/`, `data/processed/` | Data pipeline | "Data Pipeline" (raw → processed → index config) |
| `scripts/` | Operations | "Scripts" (seed, migrate, healthcheck) |
| `frontend/`, `ui/`, `app/` | UI | "Frontend" (containerized separately) |
| `tests/` | Testing | "Tests" (by domain: retrieval, cache, routing) |
| `docs/` | Documentation | "Documentation" (architecture, API ref, deployment) |

Not every project has all layers. Only create concepts for layers that actually exist. The annotations in parentheses are examples — derive the real description from file contents.

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
- Total file: 80-150 lines (including annotated tree)

### Annotated Tree View (after Quick Lookup)

After the Quick Lookup table, generate an annotated directory tree that shows the project structure with one-line descriptions for each major directory group. This provides instant visual understanding of what lives where.

Format — use `├──` tree characters with right-aligned annotations using `}` bracket grouping:

```markdown
## Directory Map

\```
project/
├── app/
│   ├── main.py
│   ├── config.py              ─ Entry, config, schemas
│   └── models.py
├── services/
│   ├── rag_pipeline.py
│   ├── semantic_cache.py      ─ Core business logic: pipeline, cache, routing
│   └── query_router.py
├── security/
│   ├── input_guard.py
│   └── output_filter.py       ─ Guard layers: input validation, output filtering
├── pipeline/
│   ├── clip_finder.py
│   └── compositor.py          ─ Video pipeline: clips, composition, captions
├── data/
│   ├── raw/
│   └── processed/             ─ Raw → processed → index config
├── tests/                     ─ Retrieval, cache, routing tests
└── CLAUDE.md                  ─ AI coding agent context
\```
```

**Tree rules:**
- Show max 3 levels deep
- Only show directories with 2+ source files (skip single-file dirs)
- Annotation goes on the last file/dir of each group, aligned with `─`
- Keep annotations under 50 characters
- Skip noise dirs (node_modules, __pycache__, .git, dist, venv)
- This section is optional for very small projects (under 10 files)

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

### Step 11: Detect duplication risk and suggest hook

After generating the index, analyze the codebase for **query/function duplication risk** — a common problem where Claude creates new functions instead of reusing existing ones.

**Risk signals to detect:**

| Signal | Threshold | Risk level |
|--------|-----------|------------|
| Multiple directories with DB/query functions | 2+ dirs with `def.*conn\|cursor\|db\|query` | HIGH |
| Same function name in different files | Any exact `def name(` match across files | CRITICAL (already duplicated) |
| Large number of utility functions (50+) across project | Count `def ` lines across source dirs | MEDIUM |
| Router files with inline DB queries (not using shared helpers) | `SELECT\|INSERT\|UPDATE\|DELETE` inside router files | MEDIUM |

**Detection method:**

```bash
# Count function definitions per watched directory
for dir in $(find . -maxdepth 2 -type d -name "routers" -o -name "api" -o -name "services" -o -name "ingestion" -o -name "pipeline" 2>/dev/null); do
  count=$(grep -rn "def " "$dir" --include="*.py" 2>/dev/null | wc -l)
  echo "$dir: $count functions"
done

# Find exact duplicate function names across directories
grep -rn "^[[:space:]]*\(async \)\?def " --include="*.py" . 2>/dev/null | \
  sed 's/.*def \([a-zA-Z_]*\)(.*/\1/' | sort | uniq -d
```

**If risk is detected (2+ signals at MEDIUM or above):**

Tell the user:

```
Duplication risk detected — [N] directories with query/DB functions, [N] duplicate function names found.

This is a known issue in larger codebases: Claude sometimes creates new functions instead of reusing existing ones, especially during complex multi-step tasks.

Recommendation: install a PostToolUse hook that greps for duplicate function names after each Edit/Write in these directories. It runs in ~90ms, costs zero API tokens, and blocks with a warning when it detects:
- Exact duplicate function names in other files
- Semantically similar names (60%+ token overlap)

Want me to generate the hook? It monitors: [list detected directories]
```

**If the user agrees**, generate a `query-duplication-checker.js` hook following this pattern:

1. **PostToolUse** on `Edit|Write` events
2. Only triggers for `.py` files in watched directories (the ones detected above)
3. Extracts new function names from `new_string` or `content`
4. Uses `execFileSync('grep', ...)` (not `exec` — no shell injection) to search for each function name specifically in other watched directories
5. For semantic similarity: splits function names into tokens (`snake_case` → tokens), checks 60%+ overlap
6. Outputs `{"decision": "block", "reason": "..."}` with file:line references when duplicates found
7. Silent exit (no block) when no duplicates
8. **Critical:** use `maxBuffer` appropriate for project size, and grep per-function (not all functions at once) to avoid ENOBUFS on large codebases
9. Exclude vendored/bundled directories (e.g., `core/` with third-party code)

Register the hook in `~/.claude/settings.json` under `PostToolUse` with matcher `Edit|Write`.

**If risk is LOW (single directory, few functions):** Skip silently — the hook overhead isn't worth it.

## Edge cases

**Monorepos:** Detect sub-apps by separate package.json/requirements.txt/go.mod. Prefix concept names with app name: "Dashboard / Auth", "API / Auth".

**Vendor/bundled code:** Directories like `core/` with 50+ subdirs that are third-party. Do NOT index internals. Mention as single concept: "Hermes Agent Core (bundled — do not modify)".

**Non-code projects:** Documentation, marketing, research projects. Group by theme: "Marketing Frameworks", "Research — AI Act", "Strategy Docs".

**No README or CLAUDE.md:** Rely entirely on directory structure + file content analysis.

**Binary/data directories:** Skip. Mention as output target in relevant concept if applicable.

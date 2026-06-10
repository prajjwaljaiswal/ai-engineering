# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **curriculum, not an app**. 20 phases, 500+ lessons teaching AI engineering from raw math up to autonomous swarms. Every algorithm is built by hand first (backprop, tokenizer, attention, agent loop) in Python, TypeScript, Rust, or Julia, then re-run through the production library. The lessons are the product.

**`AGENTS.md` is the authoritative operating manual** — read it before any contribution. It defines the hard rules (one commit per lesson, conventional commits, dependency allowlist, Mermaid/SVG-only diagrams), the lesson contract (frontmatter, `quiz.json` schema, code/test requirements), and the new-lesson onboarding flow. This file summarizes the architecture and commands; AGENTS.md governs when they conflict.

## Architecture

The repo is **content + a generated static site that renders it**. There is no application server.

```
phases/NN-phase-slug/MM-lesson-slug/
  docs/en.md        # lesson explainer (frontmatter: Type, Languages, Prerequisites, Time)
  code/main.<lang>  # hand-built implementation, runs end-to-end and exits 0
  code/tests/       # 5+ stdlib unit tests
  quiz.json         # exactly 6 questions: 1 pre + 3 check + 2 post (canonical schema only)
  outputs/          # the lesson's reusable artifact: skill-/prompt-/agent-/mcp-server-*.md
```

**Data flow — three derived surfaces that you must NOT hand-edit:**
- `README.md` (lesson tables) + `ROADMAP.md` (status) + `glossary/terms.md` are the **hand-maintained source of truth**.
- `scripts/build_catalog.py` → `catalog.json` (gitignored, rebuilt every CI run).
- `site/build.js` parses README + ROADMAP + glossary → `site/data.js` (committed by CI on push to main; the whole site is static and reads this file).
- `site/build.js` derives each lesson's GitHub URL from the **markdown link** in the README row. A plain-text README row produces a broken/missing site link — every new lesson needs `[Title](phases/NN-phase/MM-lesson/)`, not bare text.

**Aggregated outputs:** root `outputs/{skills,prompts,agents,mcp-servers}/` collect the per-lesson `outputs/` artifacts; `scripts/install_skills.py` walks the per-lesson files and installs them into a target dir.

## Commands

Validation before pushing (also gated by `.github/workflows/curriculum.yml`):

```bash
python3 scripts/audit_lessons.py            # invariant checks across all lessons — BLOCKING in CI
python3 scripts/audit_lessons.py --phase 14 # scope to one phase
python3 scripts/check_readme_counts.py      # advisory; CI auto-fixes on merge to main
python3 scripts/check_readme_counts.py --fix # apply the count fixes locally
node site/build.js                          # regenerate site/data.js (normally left to CI)
python3 scripts/lesson_run.py --phase 14    # byte-compile lesson Python; --execute to actually run
```

Run a single lesson's code + tests (from the lesson's `code/` dir):

```bash
# Python
python3 main.py && python3 -m unittest discover tests -v
# TypeScript
npx tsx main.ts && npx tsx --test
# Rust   — single-file, no Cargo project
rustc --edition 2021 main.rs && ./main
# Julia
julia main.jl
```

Scaffold a new lesson (prefer over manual mkdir): `scripts/scaffold-lesson.sh <phase-dir> <lesson-slug> [title]`.

## Conventions that bite

- **`code/main.*` must self-terminate** — no infinite stdin loops, no hangs waiting on missing API keys. `lesson_run.py` defaults to syntax-only because heavy lessons need torch/anthropic; mark such entry files with a `# requires: pkg1, pkg2` header so `--execute` skips them.
- **`quiz.json` accepts only the canonical schema** (`stage/question/options/correct/explanation`, `correct` zero-indexed). The legacy `q/choices/answer` shape crashes the site renderer silently.
- **`docs/en.md` `**Languages:**` field must match** the languages that actually have a `main.*` file in `code/`. The audit enforces this.
- **Stdlib-first dependency allowlist** (see AGENTS.md table). Rust is stdlib-only single-file; Julia uses only stdlib modules. If a tool/finding suggests a banned dep, skip it.
- **Never commit generated files**: `catalog.json` (gitignored), `site/data.js` (CI rebuilds on main), `package-lock.json` (never tracked).

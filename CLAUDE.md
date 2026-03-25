# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Continuous Claude is a persistent, learning, multi-agent development environment built on Claude Code. It transforms Claude Code into a continuously learning system that maintains context across sessions via handoffs, orchestrates specialized agents, and reduces token usage through TLDR code analysis.

**Mantra: Compound, don't compact.** Extract learnings automatically, then start fresh with full context.

## Project Layout

- **`opc/`** - Python backend (pyproject.toml lives here, all `uv` commands run from here)
  - `scripts/setup/wizard.py` - Installation wizard (12 steps)
  - `scripts/core/` - Core utilities (recall_learnings.py, store_learning.py)
  - `scripts/tldr/` - TLDR code analysis CLI
  - `docker/` - Docker Compose for PostgreSQL (pgvector)
- **`.claude/`** - Claude Code integration layer
  - `hooks/src/` - TypeScript hook sources (compiled to `dist/` via esbuild)
  - `hooks/*.py` and `hooks/*.sh` - Python/shell hooks
  - `skills/` - 109 skill definitions (SKILL.md files)
  - `agents/` - 32 agent definitions (.md and .json)
  - `rules/` - Project rules loaded into context
  - `settings.json` - Hook registrations and configuration
- **`proofs/`** - Lean 4 theorem proving files
- **`docker/`** - Docker Compose + init-schema.sql for PostgreSQL

## Build & Development Commands

### Python (from `opc/` directory)
```bash
cd opc
uv sync                          # Install dependencies
uv sync --extra dev              # Install with dev tools
uv run pytest tests/ -v          # Run all tests
uv run pytest tests/unit/ -x -v  # Run unit tests, stop on first failure
uv run ruff check .              # Lint
uv run ruff format .             # Format
uv run mypy src/                 # Type check
```

### TypeScript Hooks (from `.claude/hooks/`)
```bash
cd .claude/hooks
npm install                      # Install dependencies
npm run build                    # Build hooks (esbuild → dist/*.mjs)
npm run check                    # Type check (tsc --noEmit)
npm test                         # Run tests (vitest)
npm run dev                      # Build + type check
```

### Setup Wizard
```bash
cd opc
uv run python -m scripts.setup.wizard           # Install
uv run python -m scripts.setup.wizard --uninstall # Uninstall
```

### Docker (PostgreSQL with pgvector)
```bash
docker compose -f docker/docker-compose.yml up -d
# Default: postgresql://claude:claude_dev@localhost:5432/continuous_claude
```

## Architecture

### Three-Layer System

1. **Skills** (`.claude/skills/`) - Modular capabilities triggered by natural language or `/slash-commands`. Meta-skills like `/fix`, `/build`, `/tdd` orchestrate multi-step workflows that chain agents together.

2. **Agents** (`.claude/agents/`) - Specialized sub-sessions spawned via the Task tool. Key agents: scout (codebase exploration), oracle (external research), kraken (implementation), arbiter (testing), phoenix (refactoring analysis), sleuth (debugging).

3. **Hooks** (`.claude/hooks/`) - Intercept Claude Code lifecycle events (SessionStart, PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, Stop, SessionEnd). Registered in `.claude/settings.json`. TypeScript hooks are bundled with esbuild (zero runtime deps except better-sqlite3).

### Data Layer

- **PostgreSQL + pgvector**: 4 tables - `sessions` (cross-terminal awareness), `file_claims` (file locking), `archival_memory` (learnings with 1024-dim BGE embeddings), `handoffs` (session state)
- **File system**: `thoughts/ledgers/` (continuity ledgers), `thoughts/shared/handoffs/*.yaml` (session handoffs), `thoughts/shared/plans/*.md` (implementation plans)
- **TLDR cache**: `.tldr/` directory for code analysis artifacts

### Continuity Loop

SessionStart loads continuity ledger + recalls memories + warms TLDR cache. During work, hooks track file changes and index handoffs. PreCompact auto-generates YAML handoffs. SessionEnd triggers a daemon that extracts learnings into `archival_memory`. On `/clear`, fresh context starts with preserved state.

### Hook Pipeline

Hooks are the nervous system. Key hooks by lifecycle:
- **PreToolUse**: path-rules (enforce file boundaries), file-claims (cross-terminal locking), tldr-read-enforcer (suggest TLDR over raw reads), edit-context-inject (add surrounding code context)
- **PostToolUse**: compiler-in-the-loop (run pyright/ruff after edits), import-validator, typescript-preflight
- **UserPromptSubmit**: skill-activation-prompt (suggests relevant skills/agents), memory-awareness (surfaces past learnings), impact-refactor (warn about high-impact changes)
- **PreCompact**: auto-handoff generation
- **Stop**: auto-handoff-stop, compiler-in-the-loop-stop

## Key Conventions

- Python requires **Python 3.12+** and uses `uv` as package manager
- TypeScript hooks use **ESM** (`"type": "module"`) and build to `.mjs` files
- Hooks read JSON from stdin and output JSON to stdout
- Agent definitions use frontmatter (name, description, model, tools) followed by a system prompt
- Skills are defined in `SKILL.md` files and registered in `skill-rules.json` for trigger matching
- Environment variables go in `.env` (see `.env.example`); the canonical DB var is `CONTINUOUS_CLAUDE_DB_URL`
- The `thoughts/` directory is gitignored - it holds per-session state (ledgers, handoffs, plans)

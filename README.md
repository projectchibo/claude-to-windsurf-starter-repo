# AI Collaboration Pipeline

A skills-based pipeline for structured AI collaboration between an **architect** (designs, decomposes, documents) and an **executor** (implements). Works with Claude Code, Windsurf, Cursor, or any AI tool that reads markdown.

---

## How it works

```
Architect session (Claude Code / Windsurf in architect mode)
  ├── 1. Sync docs from completed tasks  (reads sync-queue.md)
  ├── 2. Orient using architecture.md + context files
  └── 3. Create tasks in tasks/

Executor session (Windsurf / Cursor / any AI)
  ├── 1. Pick up a pending task
  ├── 2. Load only the context listed in the task
  ├── 3. Implement to acceptance criteria
  └── 4. Mark done → write to sync-queue.md → move to tasks/done/

Repeat
```

The pipeline protocol lives in `pipeline/skills/` — decoupled from your project documentation. Bootstrap files (`CLAUDE.md`, `AGENTS.md`) are permanent stubs that point at the skills. They never grow.

---

## Roles

| | Architect | Executor |
|---|---|---|
| **Creates tasks** | ✅ | ❌ |
| **Executes tasks** | ❌ | ✅ |
| **Updates docs/context** | ✅ | ❌ (flags via sync-queue) |
| **Manages skills** | ✅ | ❌ |
| **Writes to sync-queue** | ❌ (clears it) | ✅ (appends to it) |

---

## Model Tiers

The architect selects the cheapest model that can handle the task. Break tasks down before escalating.

| Tier | Model | Used for |
|------|-------|----------|
| cheap | SWE-1.6 | Mechanical, well-defined — boilerplate, simple fixes, CRUD |
| medium | GPT-5.2 | Feature implementation, refactoring, moderate ambiguity |
| strong | claude-sonnet-4-5 / claude-opus-4-6 | Complex architecture, security-sensitive, deep reasoning |

---

## Repo Structure

```
pipeline/
  skills/                        # Edit here to update the pipeline protocol
    architect.md                 # Full architect protocol
    executor.md                  # Full executor protocol
    generate-consumer-skill.md   # Cross-repo skill generation
    create-migration.md          # Cross-repo migration protocol
    bootstrap.md                 # Repo setup and migration

.claude/commands/                # Claude Code slash commands (wrappers → pipeline/skills/)
.windsurf/skills/                # Windsurf skills (wrappers → pipeline/skills/)

docs/
  architecture.md                # System architecture (keep current)
  sync-queue.md                  # Pending doc updates for the architect to process
  context/                       # Domain-specific context files (architect-managed)
  migrations/                    # Pending cross-repo changes (architect creates, carried manually)

tasks/
  _template.md                   # Task template
  done/                          # Completed tasks

skills/
  exports/                       # Consumer skills generated for other repos

CLAUDE.md                        # Thin stub → pipeline/skills/architect.md
AGENTS.md                        # Thin stub → pipeline/skills/executor.md
```

---

## Starting a new project

### Option A — Use this repo as a template
1. Clone or copy this repo into your new project
2. Open Claude Code (or Windsurf in architect mode) and run `/bootstrap`
3. Answer the prompts: which tools you use, brief description of the project
4. The bootstrap skill creates the right config files for your tools and initializes `docs/architecture.md`
5. Start creating tasks

### Option B — Manual setup
1. Copy the `pipeline/` folder into your project
2. Create `CLAUDE.md` with:
   ```
   # Claude — Architect
   Your role is architect. Read pipeline/skills/architect.md now.
   ```
3. Create `AGENTS.md` with:
   ```
   # Windsurf — Executor
   Your role is executor. Read pipeline/skills/executor.md now.
   ```
4. Copy `.claude/commands/` and/or `.windsurf/skills/` for your tools
5. Create `docs/`, `tasks/done/`, `skills/exports/`, `docs/migrations/`

---

## Migrating an existing project

If you have a project with a monolithic `CLAUDE.md` or `AGENTS.md` that has grown to mix pipeline protocol with project-specific content:

1. Copy the `pipeline/` folder into your project
2. Open Claude Code and run `/bootstrap`
3. Choose **"Migrate existing repo"**
4. The bootstrap skill reads your existing `CLAUDE.md`/`AGENTS.md`, separates pipeline protocol from project-specific content, moves the project content into `docs/context/` and `docs/architecture.md`, and replaces the bootstrap files with thin stubs

Everything is preserved — just reorganized into the right places.

---

## Cross-repo skills

### Sharing knowledge with consuming repos

When another repo needs to understand this repo's public API or usage patterns, run `/generate-consumer-skill`. It reads your current context and generates a lean, self-contained skill file in `skills/exports/[repo-name].md`.

The consuming repo copies that file into its own `.windsurf/skills/` or `.claude/commands/` folder. Their AI can then invoke it without ever opening your repo.

Keep the export updated as your public API evolves.

### Propagating changes to dependency repos

When you realize a fix or feature belongs in a dependency rather than the current repo, run `/create-migration`. It creates a structured migration file in `docs/migrations/` that you carry to the target repo the next time you open it.

The migration file is already formatted as a task brief — it has acceptance criteria and the context the executor needs. Drop it in the target repo's `docs/migrations/` and use it to create a task.

---

## Switching the architect between tools

By default, `AGENTS.md` loads the executor role. To use Windsurf (or any tool reading `AGENTS.md`) as the architect instead, edit `AGENTS.md` to:

```
# — Architect

Your role is architect in this pipeline. Your full protocol is in pipeline/skills/architect.md.

Read it now before starting any session.
```

Switch back by restoring the executor stub. The skill content is tool-agnostic — the same protocol works regardless of which AI reads it.

---

## Updating the pipeline protocol

All protocol content lives in `pipeline/skills/`. To update it:

1. Edit the relevant file in `pipeline/skills/`
2. Copy the updated content to `.claude/commands/[skill].md` (Claude Code)
3. Copy the updated content to `.windsurf/skills/[skill]/SKILL.md` (keep the frontmatter, replace the body)

The tool-specific files are thin wrappers — their job is just to surface the skill through each tool's native interface.

---

## Tool compatibility

| Tool | Bootstrap file | Skills location | Notes |
|------|---------------|-----------------|-------|
| Claude Code | `CLAUDE.md` | `.claude/commands/` | Slash commands via `/skill-name` |
| Windsurf | `AGENTS.md` | `.windsurf/skills/` | Auto-invoked by Cascade or via `@skill-name` |
| Cursor | `.cursorrules` | n/a | Point at `pipeline/skills/executor.md` directly |
| Other | `PIPELINE.md` | n/a | Drop the skill content in a file your tool reads |

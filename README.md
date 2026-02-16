# lw-workflow

A Claude Code plugin providing a complete TDD development workflow with Linear integration, agent team orchestration, and Railway deployment. 14 skills and 3 agents for planning, implementing, reviewing, auditing, and releasing TypeScript projects.

## Installation

Add the plugin to any Claude Code project:

```bash
claude --add-dir /path/to/claude-workflow-plugin
```

Or clone and reference:

```bash
git clone https://github.com/lucaswall/claude-workflow-plugin.git
claude --add-dir ~/claude-workflow-plugin
```

Skills are invoked with the plugin namespace: `/lw-workflow:plan-inline`, `/lw-workflow:code-audit`, etc.

## Requirements

- **Claude Code** with skills and agents support
- **Linear** account with MCP configured (issue tracking)
- **GitHub** with `gh` CLI (PRs)
- **Node.js** project with `npm test`, `npm run lint`, `npm run build`
- **Agent teams** require `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json

Optional:
- **Railway** MCP for deployment management
- **Playwright** for E2E tests (`npm run e2e`)
- **Drizzle ORM** for database migration management

## Project Compatibility

Designed for TypeScript projects that follow these patterns:

- **TypeScript** (strict mode) with any framework (Next.js, Fastify, Express, etc.)
- **Linear** for issue tracking (Backlog → Todo → In Progress → Review → Merge → Done → Released)
- **Railway** for deployment (optional, discovered from CLAUDE.md)
- **PostgreSQL + Drizzle ORM** for database (optional)
- **Vitest** or Jest for unit testing, Playwright for E2E
- **TDD workflow** (test-first development)

### Tested Archetypes

| Archetype | Stack | Example |
|-----------|-------|---------|
| **Frontend app** | Next.js App Router, Tailwind/shadcn, OAuth, PWA | Food logging app with AI analysis |
| **Backend API** | Fastify, Google Drive/Sheets, Apps Script/clasp | Invoice management system |

Both archetypes share: TypeScript strict, Railway, Linear, PostgreSQL + Drizzle, Vitest, pino logging.

## CLAUDE.md Integration

Skills discover project context from your `CLAUDE.md` file. They look for:

- **LINEAR INTEGRATION** section: team name, issue prefix, state workflow
- **STRUCTURE** section: file organization, test conventions
- **ENVIRONMENTS** section: deployment targets, branches, URLs
- **COMMANDS** section: test, lint, build commands
- **MCP SERVERS** section: available MCP tools

If sections are missing, skills fall back to dynamic discovery (e.g., `mcp__linear__list_teams`) or general conventions.

## Skills

### Planning

| Skill | Trigger | Description |
|-------|---------|-------------|
| **plan-inline** | `/plan-inline <task>` | Create TDD plan from direct request. Creates Linear issues in Todo. |
| **plan-fix** | `/plan-fix <bug>` | Investigate bug + create fix plan. Creates Linear issue in Todo. |
| **plan-backlog** | `/plan-backlog [PROJ-123]` | Convert Linear Backlog issues to TDD plan. Moves to Todo. |
| **add-to-backlog** | `/add-to-backlog <ideas>` | Parse free-form input into Linear Backlog issues. |
| **backlog-refine** | `/backlog-refine PROJ-123` | Interactive refinement of vague Backlog issues. |

### Implementation

| Skill | Trigger | Description |
|-------|---------|-------------|
| **plan-implement** | `/plan-implement` | Execute PLANS.md with agent team (parallel workers). TDD loop. |
| **plan-review-implementation** | `/plan-review-implementation` | QA review with 3-reviewer agent team. Moves issues Review → Merge. |

### Auditing

| Skill | Trigger | Description |
|-------|---------|-------------|
| **code-audit** | `/code-audit [area]` | Full codebase audit with 3-reviewer team. Creates Backlog issues. |
| **frontend-review** | `/frontend-review [area]` | Frontend audit with 4-reviewer team (a11y, UX, perf, visual QA). |
| **deep-review** | `/deep-review <feature>` | Deep cross-domain analysis of a single screen/feature. |

### Research & Investigation

| Skill | Trigger | Description |
|-------|---------|-------------|
| **investigate** | `/investigate <issue>` | Read-only investigation. Reports findings without creating plans. |
| **pull-from-roadmap** | `/pull-from-roadmap <feature>` | Deep research + interactive discussion of a feature idea. |

### Release

| Skill | Trigger | Description |
|-------|---------|-------------|
| **push-to-production** | `/push-to-production` | Full release: backup DB, migrate, merge to release branch. |

### Meta

| Skill | Trigger | Description |
|-------|---------|-------------|
| **tools-improve** | `/tools-improve` | Best practices for creating/modifying skills, agents, CLAUDE.md. |

## Agents

| Agent | Model | Permission | Purpose |
|-------|-------|------------|---------|
| **verifier** | Haiku | dontAsk | Test + lint + build validation (3 modes: TDD, Full, E2E) |
| **pr-creator** | Sonnet | bypassPermissions | Full PR workflow with Linear integration |
| **bug-hunter** | Sonnet | dontAsk | OWASP-based code review of git changes |

## Workflows

### Feature from roadmap
```
pull-from-roadmap → add-to-backlog → backlog-refine (optional)
  → plan-backlog → plan-implement → plan-review-implementation (repeat)
  → push-to-production
```

### Direct request
```
plan-inline → plan-implement → plan-review-implementation (repeat)
  → push-to-production
```

### Bug fix
```
investigate (optional) → plan-fix → plan-implement
  → plan-review-implementation (repeat) → push-to-production
```

### Audit-driven
```
code-audit / frontend-review / deep-review
  → backlog-refine (optional) → plan-backlog → plan-implement
  → plan-review-implementation (repeat) → push-to-production
```

## Agent Teams

Skills that use agent teams (`plan-implement`, `plan-review-implementation`, `code-audit`, `frontend-review`) automatically:

1. Create a team with `TeamCreate`
2. Spawn specialized workers/reviewers (Sonnet model)
3. Coordinate via shared task list and messaging
4. Merge findings and handle Linear updates (lead only)
5. Clean up with `TeamDelete`

All team skills include a **single-agent fallback** if `TeamCreate` fails.

### Key constraints

- **File ownership**: No file assigned to multiple workers
- **Lead handles MCP**: Workers cannot reliably access MCP tools
- **Cap at 4 workers**: Diminishing returns beyond that
- **Workers use TDD-mode verifier only**: Full/E2E verifier is lead-exclusive

## Reference Files

Skills include detailed reference checklists:

| Skill | Reference Files |
|-------|----------------|
| code-audit | compliance-checklist.md, category-tags.md, priority-assessment.md |
| plan-review-implementation | code-review-checklist.md, reviewer-prompts.md |
| frontend-review | frontend-checklist.md, reviewer-prompts.md |
| deep-review | deep-review-checklist.md |
| push-to-production | migration-example.md, changelog-guidelines.md |
| tools-improve | claude-md-reference.md, skills-reference.md, subagents-reference.md, agent-teams-reference.md |

## File Structure

```
claude-workflow-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── agents/
│   ├── verifier.md              # Test/build runner (Haiku)
│   ├── pr-creator.md            # PR creator (Sonnet)
│   └── bug-hunter.md            # Code reviewer (Sonnet)
├── skills/
│   ├── add-to-backlog/          # Free-form → Linear Backlog
│   ├── backlog-refine/          # Interactive issue refinement
│   ├── code-audit/              # Codebase audit (agent team)
│   │   └── references/          # Audit checklists
│   ├── deep-review/             # Focused feature analysis
│   │   └── references/          # Cross-domain checklist
│   ├── frontend-review/         # Frontend audit (agent team)
│   │   └── references/          # Frontend + reviewer checklists
│   ├── investigate/             # Read-only investigation
│   ├── plan-backlog/            # Backlog → TDD plan
│   ├── plan-fix/                # Bug → fix plan
│   ├── plan-implement/          # Execute plan (agent team)
│   ├── plan-inline/             # Direct request → TDD plan
│   ├── plan-review-implementation/  # QA review (agent team)
│   │   └── references/          # Review checklists
│   ├── pull-from-roadmap/       # Feature research + discussion
│   ├── push-to-production/      # Release automation
│   │   └── references/          # Migration + changelog guides
│   └── tools-improve/           # Meta: skill/agent best practices
│       └── references/          # Claude Code extensibility docs
├── LICENSE
└── README.md
```

## Migrating Skills Into Your Project

Instead of installing lw-workflow as a shared plugin, you can migrate skills directly into your project's `.claude/` directory. This makes them part of your codebase, removes the namespace prefix, and lets you customize them for your specific stack.

### Why Migrate

- **Simpler invocation** — `/plan-inline` instead of `/lw-workflow:plan-inline`
- **Project-specific** — Skills adapt to your exact stack, tools, and conventions
- **Version controlled** — Skills evolve alongside your project
- **Selective adoption** — Pick only the skills you need

### How To Migrate

Clone the repo inside your target project:

```bash
cd your-project
git clone https://github.com/lucaswall/claude-workflow-plugin.git
```

Then open Claude Code and use this prompt:

```
Read all skills and agents from claude-workflow-plugin/:
- skills/*/instructions.md, skills/*/metadata.yml, skills/*/references/*.md
- agents/*.md
- Also read skills/tools-improve/references/ — they document how skills, agents,
  and agent teams are structured.

Then study THIS project: read CLAUDE.md, understand the architecture, tech stack,
test/lint/build commands, deployment targets, and issue tracking setup.

Migrate the skills and agents into this project's .claude/ directory, adapting them
to fit this project:

1. Replace integrations: swap Linear/Railway/Drizzle references with whatever this
   project uses (or remove sections for tools we don't have).
2. Adapt commands: match our test runner, linter, build tool, and package manager.
3. Adapt conventions: adjust file paths, naming patterns, and framework idioms to
   match our codebase structure.
4. Preserve the workflow patterns: keep TDD loop, agent team orchestration,
   review checklists, and PLANS.md coordination — just adapt the specifics.
5. Keep stack-agnostic references (review checklists, priority assessment, etc.)
   and adapt or drop stack-specific ones.
6. Create skills under .claude/skills/ and agents under .claude/agents/.
7. Add attribution: include a comment in CLAUDE.md or in each migrated skill:
   "Adapted from lw-workflow by Lucas Wall (CC BY 4.0)
   https://github.com/lucaswall/claude-workflow-plugin"

Ask me before starting if anything about our stack or workflow is unclear.
```

After migration, delete the cloned repo:

```bash
rm -rf claude-workflow-plugin
```

This plugin is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — you are free to copy, adapt, and use the skills in any project, including commercially, as long as you give appropriate credit.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) - Lucas Wall <wall.lucas@gmail.com>

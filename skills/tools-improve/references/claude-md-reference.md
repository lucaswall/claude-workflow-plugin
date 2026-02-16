# CLAUDE.md Best Practices

Source: [Anthropic official docs](https://code.claude.com/docs/en/best-practices) and [memory docs](https://code.claude.com/docs/en/memory).

## Core Principle

CLAUDE.md is loaded into **every session**. It consumes context that Claude needs for actual work. Every line must earn its place.

> "For each line, ask: 'Would removing this cause Claude to make mistakes?' If not, cut it."
> "Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"

## What to Include

| Category | Examples |
|---|---|
| Bash commands Claude can't guess | Build, test, lint commands with non-standard flags |
| Code style rules **that differ from defaults** | `@/` path aliases, unconventional patterns |
| Testing instructions | Preferred runners, test file conventions |
| Repository etiquette | Branch naming, PR conventions, commit rules |
| Architectural decisions | Non-obvious design choices specific to the project |
| Developer environment quirks | Required env vars, special setup steps |
| Common gotchas | Non-obvious behaviors that cause repeated mistakes |

## What to Exclude

| Category | Why | Alternative |
|---|---|---|
| File-by-file codebase descriptions | Claude discovers files via Glob/Grep | Directory-level overview only |
| Anything Claude can infer from code | Wastes context on redundant info | Let Claude read the code |
| Standard language conventions | Claude already knows them | Only document deviations |
| Detailed API documentation | Changes frequently, large | Link to source files or docs |
| Information that changes frequently | Goes stale, misleads Claude | Use `@imports` or let Claude read source |
| Long explanations or tutorials | Bloats context | Move to skills or reference files |
| Self-evident practices | "Write clean code" adds nothing | Delete |

## Modular Organization

For larger projects, use `.claude/rules/*.md` for topic-specific instructions:
- Each file covers one topic (e.g., `testing.md`, `api-design.md`)
- Supports path-scoping via YAML `paths` frontmatter
- All loaded automatically as project memory
- Subdirectories supported (discovered recursively)

Use `@path/to/file` imports in CLAUDE.md to reference external files (up to 5 levels deep).

## When Reviewing CLAUDE.md

Apply this checklist:

1. **File-by-file structure tree?** → Replace with directory-level overview or remove entirely
2. **Style rules Claude already knows?** → Remove (camelCase for JS, PascalCase for components, etc.)
3. **Full API endpoint table?** → Replace with pointer to route files, keep only non-obvious formats
4. **Full env var listing?** → Replace with pointer to `.env.sample`, keep only gotchas
5. **Detailed config sections?** → If Claude can read the config file, just point to it
6. **Domain knowledge only relevant sometimes?** → Move to a skill
7. **Stale content?** → Cross-check tables/lists against actual files (agents, DB tables, components)
8. **Emphasis on critical rules?** → Use "IMPORTANT" or "YOU MUST" for rules that must not be ignored
9. **Checked into git?** → CLAUDE.md should be in version control for team sharing

## When Adding to CLAUDE.md

Before adding a new line, ask:
1. Would Claude make mistakes without this? If no → don't add
2. Can Claude figure this out by reading code? If yes → don't add
3. Is this relevant to every session? If no → put it in a skill or `.claude/rules/`
4. Is this a standard convention? If yes → don't add (only document deviations)
5. Will this go stale? If yes → point to the source file instead

## CLAUDE.md vs Skills vs Rules

| Content | Where |
|---|---|
| Project-wide conventions (every session) | CLAUDE.md |
| Path-specific rules (e.g., API files only) | `.claude/rules/` with `paths` frontmatter |
| Domain workflows (invoked on demand) | `.claude/skills/` |
| Background knowledge (auto-loaded when relevant) | Skill with `user-invocable: false` |
| Personal preferences (not shared) | `CLAUDE.local.md` or `~/.claude/CLAUDE.md` |

---
name: tools-migrate
description: "Migrate skills and agents between this plugin and another project. Use when: syncing workflow tools to a target project, pulling improvements back from a project, comparing skill versions, or adapting tools to a different architecture. Supports bidirectional migration with architecture adaptation."
argument-hint: <to|from> <project-path>
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, Task, WebSearch, WebFetch, AskUserQuestion
disable-model-invocation: true
---

# Tools Migrate — Bidirectional Skill & Agent Migration

Migrate skills and agents between this plugin (`lw-workflow`) and a target project directory. Adapts tools to the target's architecture, infrastructure, and conventions.

## Directions

| Direction | Syntax | Purpose |
|-----------|--------|---------|
| **to** | `/tools-migrate to <project-path>` | Push skills/agents from this plugin to a target project |
| **from** | `/tools-migrate from <project-path>` | Pull improvements from a project back into this plugin |

## Argument Parsing

Parse `$ARGUMENTS` to extract:
- **Direction:** first word — `to` or `from`
- **Project path:** remaining text — absolute or relative path to the target project

If direction or path is missing/ambiguous, ask the user before proceeding.

Resolve the project path to an absolute path. Verify it exists and contains a project (look for `package.json`, `CLAUDE.md`, `.git`, or similar markers). If the path doesn't exist, STOP and tell the user.

## Phase 1: Discovery

Perform all discovery in parallel where possible.

### 1A. Inventory This Plugin

Read the plugin manifest and enumerate all skills and agents:

**Skills** — For each directory under `skills/`:
- Read `SKILL.md` frontmatter (name, description, allowed-tools)
- Note the number of reference files
- Get last modified date: `git log -1 --format=%ai -- skills/<name>/`

**Agents** — For each file under `agents/`:
- Read frontmatter (name, description, model, permissionMode)
- Get last modified date: `git log -1 --format=%ai -- agents/<name>.md`

### 1B. Inventory Target Project

Read the target project's structure:

1. **CLAUDE.md** — Read fully. Extract:
   - Project name, description, tech stack
   - Available MCP servers and integrations (LINEAR INTEGRATION, MCP SERVERS, etc.)
   - Build/test/lint commands (COMMANDS section)
   - Project structure (STRUCTURE, FOLDER STRUCTURE sections)
   - Environments and deployment setup (ENVIRONMENTS section)
   - Any existing skill/agent tables or references

2. **Existing skills** — Glob for `.claude/skills/*/SKILL.md` in the target project
   - Read each SKILL.md frontmatter
   - Get last modified date from git if available

3. **Existing agents** — Glob for `.claude/agents/*.md` in the target project
   - Read each agent frontmatter
   - Get last modified date from git if available

4. **Package/config files** — Read `package.json`, `tsconfig.json` to understand:
   - Language (TypeScript, JavaScript, Python, etc.)
   - Framework (Next.js, Express, Fastify, etc.)
   - Test runner (vitest, jest, pytest, etc.)
   - Linter/formatter configuration

### 1C. Change History

For skills/agents that exist in both locations, use git log to understand the evolution of changes on each side. This is critical for understanding *why* things diverged, not just *what* differs.

**Plugin side** (always available):
1. **Git log per skill/agent** — For each matching skill/agent:
   ```
   git log --oneline --since="<target-last-modified>" -- skills/<name>/
   ```
   This shows commits in the plugin since the target's version was last updated.
2. **Commit message analysis** — Read commit messages to understand the intent behind changes (bug fixes, new features, refactors). This helps decide whether updates are worth migrating.

**Target project side** (if git available):
1. **Git log per skill/agent** — For each matching skill/agent:
   ```
   git -C <target-path> log --oneline -- .claude/skills/<name>/ .claude/agents/<name>.md
   ```
   This shows the full history of how the target's version evolved — revealing local adaptations, bug fixes, or improvements made in-context.
2. **Commit message analysis** — Understand whether changes were intentional customizations (should be preserved) or drift from the plugin source (may need realignment).

**Content diff** — For each matching skill/agent, use `diff` or read both files to identify:
- Structural changes (new sections, removed sections)
- Instruction changes (workflow updates, new rules)
- Tool/permission changes (new allowed-tools, model changes)
- Reference file additions or removals

**Interpreting history for migration decisions:**
- Target has local fixes/improvements not in plugin → candidate for `from` backport
- Plugin has updates the target hasn't received → candidate for `to` update
- Both sides changed the same section independently → flag as diverged, needs user decision
- Target intentionally removed a section → preserve that decision, don't re-add

### 1D. Research (as needed)

Use `WebSearch` and `WebFetch` to look up documentation when:
- The target project uses frameworks/tools this plugin's skills reference that need adaptation
- You need to understand a deployment platform's CLI or MCP integration
- You need current best practices for adapting patterns to a different stack
- The target project uses infrastructure you're not familiar with

## Phase 2: Analysis

### For `to` Direction (Plugin → Project)

Classify each plugin skill/agent into one of these categories:

| Category | Criteria | Action |
|----------|----------|--------|
| **New** | Doesn't exist in target | Evaluate relevance, propose adding |
| **Update Available** | Exists in target but plugin has newer changes | Show diff summary, propose update |
| **Up to Date** | Exists in target, no changes since last sync | Skip (mention in report) |
| **Not Applicable** | Skill depends on infrastructure the target lacks | Skip with explanation |
| **Needs Adaptation** | Applicable but requires changes for target arch | Propose with adaptation details |

**Relevance Assessment for New Skills:**

| Skill Type | Relevant When Target Has... |
|------------|----------------------------|
| Planning skills (plan-inline, plan-backlog, plan-fix, etc.) | Any issue tracker (Linear, Jira, GitHub Issues) |
| Implementation skills (plan-implement, plan-review-implementation) | Test runner + linter configured |
| Audit skills (code-audit, frontend-review, deep-review) | Source code to audit |
| Investigation (investigate) | Always relevant |
| Release (push-to-production) | Deployment pipeline |
| Backlog (add-to-backlog, backlog-refine, pull-from-roadmap) | Issue tracker |
| Meta (tools-improve) | Always relevant |
| Migration (tools-migrate) | Should NOT be migrated (stays in plugin only) |

**Adaptation Analysis:**

For each skill marked "Needs Adaptation", identify:
- MCP tool name changes (e.g., `mcp__linear__*` → `mcp__jira__*` or GitHub Issues API)
- Build/test/lint command changes (e.g., `npm test` → `bun test`, `pytest`)
- Framework-specific patterns (e.g., Next.js → Express route conventions)
- File path conventions (e.g., `src/` → `app/`, test file co-location vs separate directory)
- Agent model/tool adjustments for different project needs

### For `from` Direction (Project → Plugin)

For each skill/agent in the target project:

| Category | Criteria | Action |
|----------|----------|--------|
| **Improvement** | Target has a better version of a plugin skill | Show diff, propose backport |
| **New Capability** | Target has a skill the plugin doesn't | Evaluate for generalization |
| **Project-Specific** | Target skill is too specific to generalize | Skip with explanation |
| **Diverged** | Both changed independently | Show both versions, ask user |

**When evaluating improvements:**
- Compare instruction quality (more thorough? better error handling?)
- Check for new workflow steps or patterns
- Look for bug fixes or edge case handling
- Identify new reference docs that could benefit the plugin
- Note any project-specific hardcoding that would need to be generalized

## Phase 3: Migration Proposal

Present a structured proposal to the user and start an interactive conversation.

### Proposal Format

```
## Migration Proposal: [to|from] [project-name]

### Summary
- Direction: [to project / from project]
- Plugin skills: [count] | Plugin agents: [count]
- Target skills: [count] | Target agents: [count]
- Actions proposed: [count new] | [count updates] | [count skipped]

### Proposed Actions

#### New (will be created)
1. **[skill-name]** — [why it's relevant] → [any adaptations needed]

#### Updates (newer version available)
1. **[skill-name]** — [what changed] → [adaptation notes]

#### Skipped
1. **[skill-name]** — [reason: up to date | not applicable | project-specific]

### Adaptations Required
[For each skill needing adaptation, explain what changes and why]

### Questions
[Any decisions that need user input]
```

### Interactive Conversation

After presenting the proposal, engage in conversation with the user:

1. **Confirm scope** — Which skills/agents to migrate, which to skip
2. **Resolve adaptations** — For each skill needing changes, confirm the approach
3. **Address questions** — Answer user questions about any proposed changes
4. **Handle edge cases** — Discuss skills where the right action isn't clear

Use `AskUserQuestion` for structured choices. Use regular conversation for open-ended discussion.

Do NOT proceed to execution until the user explicitly confirms.

## Phase 4: Execution

### For `to` Direction

For each confirmed skill:

1. **Create directory** — `<target-project>/.claude/skills/<name>/`
2. **Write adapted SKILL.md** — Apply all confirmed adaptations:
   - Replace MCP tool names in `allowed-tools` and instructions
   - Update command references (test, build, lint)
   - Adjust file path conventions
   - Update framework-specific patterns
   - Preserve the skill's core logic and workflow
3. **Copy/adapt references** — Process each reference file similarly
4. **Write adapted agents** — `<target-project>/.claude/agents/<name>.md`

For each confirmed agent:
1. **Write adapted agent file** — Apply model, tool, and permission adaptations

After all files are written:
5. **Update target CLAUDE.md** — If it has a skills/agents table, update it. If not, suggest adding one.
6. **Verify** — Read back each written file to confirm it was written correctly

### For `from` Direction

For each confirmed improvement:

1. **Update plugin skill** — Edit the SKILL.md in this repository
2. **Update references** — Add, modify, or remove reference files
3. **Update plugin agents** — Edit agent files in this repository
4. **Generalize** — Remove project-specific references, use dynamic discovery patterns

After all files are updated:
5. **Update plugin CLAUDE.md** — Update the skill/agent tables if they changed
6. **Verify** — Read back each modified file

### Adaptation Patterns

When adapting skills to a different tech stack, apply these transformations:

**Issue Tracker Adaptation:**
- Linear → Jira: Change `mcp__linear__*` tool names, adjust state names, update field mappings
- Linear → GitHub Issues: Use `gh` CLI instead of MCP tools, map labels/milestones
- If no issue tracker: Remove issue creation steps, keep planning output in PLANS.md only

**Test Runner Adaptation:**
- vitest/jest → pytest: Change file patterns, assertion syntax references, config file names
- Adjust verifier agent commands accordingly

**Deployment Adaptation:**
- Railway → Vercel/AWS/GCP: Change MCP tool names, adjust deployment commands
- If no deployment MCP: Remove deployment checks, simplify push-to-production

**Language Adaptation:**
- TypeScript → Python: Change file extensions in Glob patterns, adjust import conventions
- TypeScript → Go/Rust/etc.: Significant restructuring may be needed — flag for user review

**Framework Adaptation:**
- Next.js → Express/Fastify: Change route conventions, middleware patterns
- React → Vue/Svelte: Change component conventions in frontend-review

## Error Handling

| Situation | Action |
|-----------|--------|
| Target path doesn't exist | STOP — ask user to verify path |
| Target has no CLAUDE.md | Continue — use package.json and file structure for context |
| Target uses unknown framework | Use WebSearch to research, ask user if unclear |
| Skill has no equivalent in target | Mark as "New" and let user decide |
| Conflicting changes (from direction) | Show both versions, ask user to choose or merge |
| Write permission denied | STOP — tell user about the permission issue |
| Git not available in target | Skip history comparison, use file modification dates |
| MCP tools in skill don't exist in target | Flag in adaptation, suggest alternatives or removal |

## Rules

- **Never migrate `tools-migrate` itself** — This skill stays in the plugin only
- **Never overwrite without confirmation** — Always show what will change before writing
- **Preserve target customizations** — When updating, merge changes rather than overwrite
- **Generalize when pulling back** — Remove project-specific references when migrating to plugin
- **Adapt, don't copy** — Skills must work in the target's architecture
- **Research when uncertain** — Use WebSearch for framework/tool documentation
- **One direction per invocation** — Either `to` or `from`, not both

## Termination

After execution completes, output:

```
## Migration Complete

**Direction:** [to|from] [project-name]
**Skills migrated:** [count] ([count] new, [count] updated)
**Agents migrated:** [count] ([count] new, [count] updated)
**Skipped:** [count] ([reasons])

### Changes Made
- [list of files created/modified with brief description]

### Manual Steps Required
- [any steps the user needs to take, e.g., "Connect Linear MCP in target project"]
- [e.g., "Review adapted push-to-production for your deploy commands"]

### Recommendations
- [suggestions for getting the most out of migrated tools]
```

Do not offer to do additional work. Output the report and stop.

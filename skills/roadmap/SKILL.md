---
name: roadmap
description: Deep research and discussion of a roadmap feature or new idea. Gathers extensive context from codebase, web, APIs, MCPs, and project history, then presents a concise analysis report for interactive discussion. After discussion, handles the outcome — write a feature spec to the roadmap, move to backlog, create an inline plan, modify, or drop. Use when user says "roadmap", "pull from roadmap", "push to roadmap", "add to roadmap", "analyze this feature", "research this idea", or wants to evaluate a feature.
argument-hint: <roadmap item name or new feature description>
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, WebSearch, WebFetch, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Deep research, interactive discussion, and action on a feature idea. You gather extensive context, present findings, discuss with the user, then execute whatever outcome they decide — write to roadmap, pull to backlog, plan, modify, or drop.

ultrathink

## Phase 1: Input Resolution

1. **Parse $ARGUMENTS** — Determine if this is:
   - **Existing roadmap item:** A reference to a feature in `ROADMAP.md` (or similar roadmap file discovered from CLAUDE.md)
   - **New idea:** A description of something NOT currently in the roadmap

2. **Read the roadmap file** — Look for `ROADMAP.md` in the project root (or discover the roadmap file path from CLAUDE.md). If found:
   - Read the full file including its Conventions section (if any)
   - Search for a matching section by exact heading match or keyword overlap
   - If multiple matches, ask user to clarify
   - If no match found, treat as a new idea
   - Check existing features for overlap or conflict
   - Note the file's structure and conventions for later use

3. **Extract the feature spec** — If existing item, extract the full section (Problem, Goal, Design, Architecture, Edge Cases, Implementation Order). If new idea, use $ARGUMENTS as the raw description.

4. **Read CLAUDE.md** — Load project context, tech stack, conventions. Extract the team name from the LINEAR INTEGRATION section if it exists.

## Phase 2: Deep Research

Launch parallel research to build comprehensive context. Use Task agents for independent research streams. Launch all independent streams simultaneously.

### Research Stream 1: Codebase Analysis

Use Task with `subagent_type=Explore` (thoroughness: "very thorough"). Explore the codebase for everything related to this feature:
- **Current implementation** — What exists today that's relevant? What would this feature touch?
- **Architecture** — How does the current system work in the affected areas?
- **Patterns** — What conventions and patterns are established?
- **Tests** — What test coverage exists in related areas?
- **Dependencies** — What libraries/APIs are already in use that relate?

### Research Stream 2: External Research

Use Task with `subagent_type=general-purpose` and `model=opus`. Search the web for technical context:
- **API feasibility** — If the feature involves external APIs, research actual capabilities, pricing, limitations, regional coverage
- **Technical approaches** — How have others solved this? What are the trade-offs?
- **Gotchas** — Known issues, limitations, or surprises others have encountered
- **User context relevance** — Use project context from CLAUDE.md (target audience, deployment region, scale, constraints) to evaluate feasibility through the user's actual lens

**CRITICAL:** Look for real user experiences, developer forums, actual API responses — not just documentation promises or marketing pages. Evidence over claims.

### Research Stream 3: Project Context

Use Task with `subagent_type=Explore` or direct tool calls. Check project state:
- **Linear issues** — Query existing Backlog/Todo/In Progress issues for related or overlapping work (if Linear MCP available)
- **Roadmap dependencies** — Does this feature depend on or block other roadmap items?
- **Recent changes** — Any recent commits or PRs that affect this area?

### Research guidelines

- Launch independent streams in parallel for speed
- Each stream should return specific, evidence-based findings
- Include any relevant MCP context (deployment state, external API integrations) if they inform feasibility
- If the feature involves UI, examine existing component patterns and relevant pages

## Phase 3: Analysis Report

After all research completes, synthesize findings into a concise report. Output directly to the conversation:

```
## Feature Analysis: [Feature Name]

### Current State
[What exists today in the codebase that's relevant. 2-3 sentences.]

### Key Findings
[Numbered list of the most important discoveries. Each finding should be specific and evidence-backed.]

### Feasibility Assessment
[Honest assessment of whether this feature is feasible, practical, and valuable. Include regional, technical, or cost concerns.]

### Design Considerations
[Key decisions that need to be made. What are the main approaches? What trade-offs exist?]

### Recommendations
[What you recommend — implement as-is, modify approach, add to roadmap, defer, drop, or split. Explain why.]

### Open Questions
[Genuine questions for the user that would affect the decision or feature design.]
```

**Keep it concise.** Details were gathered for YOUR reasoning. The user gets insights and conclusions.

## Phase 4: Interactive Discussion

After presenting the report, the conversation continues naturally:

- Answer questions with specific evidence from the research
- Explore alternative approaches if the user pushes back
- Do additional targeted research if new angles come up
- Help the user shape the feature — refine the problem, goal, design, and edge cases through conversation
- Take note of decisions made during discussion — these feed into whatever action follows

**Do NOT rush to a conclusion.** The discussion ends when the user explicitly indicates a decision:
- "Add it to the roadmap" / "write it up" → proceed to **Write to Roadmap**
- "Add it to the backlog" → proceed to **Pull to Backlog**
- "Let's plan this" / "make an inline plan" → proceed to **Create Inline Plan**
- "Drop it" / "not worth it" → proceed to **Drop**
- "Modify the roadmap item" → proceed to **Modify**
- "Let me think about it" → stop, no action needed
- "Actually, add this to the backlog instead" → tell the user to run `/add-to-backlog` with the refined description

## Phase 5: Action

When the user decides on an action, execute the matching path:

### Write to Roadmap

When the user wants to add a new feature to the roadmap:

1. **Read the roadmap file** to get current content and conventions.

2. **Draft the feature spec** following the roadmap file's conventions. The standard structure is:

   ```
   ## Feature Name

   ### Problem

   [What's wrong or missing. 2-3 sentences. Problem-focused, no solution language.]

   ### Prerequisites

   [Other features or Linear issues that must be done first. Omit section entirely if none.]

   ### Goal

   [What the feature achieves for the user. 1-2 sentences.]

   ### Design

   [The core: UX flows, behavior rules, UI details. Use sub-sections as needed for clarity.]

   ### Architecture

   [Technical decisions: storage, APIs, state management. Omit if purely UI.]

   ### Edge Cases

   [Non-obvious scenarios and how to handle them.]

   ### Implementation Order

   [Numbered list of steps, ordered by dependency.]
   ```

3. **Incorporate discussion decisions.** Everything agreed during Phase 4 goes into the spec. Do not invent details that weren't discussed — if something wasn't covered, either ask now or leave the section appropriately scoped.

4. **Present the draft** to the user in the conversation before writing. Show the full feature spec as markdown. Ask: **"Does this capture what we discussed? Any changes before I write it to the roadmap?"**

5. **After user approval**, write the feature to the roadmap file:
   - Insert the feature section before the Conventions section (or at the end if no Conventions section)
   - Add a `---` separator before the new feature (matching existing style)
   - Add a row to the Contents table with a linked feature name and one-sentence summary
   - Generate a stable slug anchor from the feature heading (lowercase, hyphens, never changes)
   - Check for cross-references: if existing features relate, add links where appropriate

6. **Verify** — Read the file after writing to confirm clean structure (no orphaned separators, no broken links, contents table in sync).

### Pull to Backlog

1. Verify Linear MCP: call `mcp__linear__list_teams`. If unavailable, STOP: "Linear MCP not connected. Run `/mcp` to reconnect."
2. Create Backlog issues following the add-to-backlog patterns (problem-focused descriptions, proper labels and priority)
3. If this was an existing roadmap item, ask: **"Remove this feature from the roadmap file?"**
4. If confirmed → run roadmap cleanup procedure

### Create Inline Plan

1. Summarize what was decided during the discussion
2. Tell the user: "Run `/plan-inline [summary]` to create the implementation plan."
3. If this was an existing roadmap item, ask: **"Remove this feature from the roadmap file?"**
4. If confirmed → run roadmap cleanup procedure

### Drop

1. If this was an existing roadmap item, ask: **"Remove this feature from the roadmap file?"**
2. If confirmed → run roadmap cleanup procedure

### Modify

1. Edit the feature section in the roadmap file with the agreed changes
2. Do NOT remove — the feature stays for future evaluation

### Roadmap cleanup procedure

When removing a feature from the roadmap:
1. Read the roadmap file to get current content
2. Delete the entire feature section (from `## Heading` through the `---` separator after it)
3. Remove the feature's row from the Contents table at the top (if one exists)
4. Check remaining features for cross-references to the removed feature — update or remove them
5. Verify file structure is clean (no orphaned separators, no broken links)
6. Follow the roadmap file's conventions section for all modifications (if one exists)

## Rules

- **Evidence over opinions** — Every finding must be backed by specific evidence (code paths, API docs, forum posts, data points)
- **Honest about uncertainty** — If you can't determine something, say so
- **User's context matters** — Use project context from CLAUDE.md (audience, region, scale, constraints) to inform the analysis
- **Don't oversell or undersell** — Present findings neutrally, let the user decide
- **Roadmap conventions** — Follow the roadmap file's conventions section for all modifications (if one exists)
- **No implementation** — This skill researches, discusses, and manages roadmap/backlog. It does NOT write code or create implementation plans (except when creating backlog issues)
- **Concise reports** — Research is thorough, output is scannable
- **Draft before writing** — Always show the user the complete feature spec and get approval before modifying the roadmap file
- **Problem-focused writing** — Problem and Goal sections use user-facing language. Technical details belong in Architecture
- **No code in specs** — Reference file paths and patterns, but don't write implementation code in the roadmap

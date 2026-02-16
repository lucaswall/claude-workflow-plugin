---
name: pull-from-roadmap
description: Deep research and discussion of a roadmap feature or new idea. Gathers extensive context from codebase, web, APIs, MCPs, and project history, then presents a concise analysis report with insights, recommendations, and questions for interactive discussion. After discussion, handles roadmap cleanup if the feature moves to backlog or implementation. Use when user says "pull from roadmap", "analyze this feature", "research this idea", or wants to evaluate a feature before committing.
argument-hint: <roadmap item name or new feature description>
allowed-tools: Read, Edit, Glob, Grep, Task, Bash, WebSearch, WebFetch, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Deep research and interactive discussion of a feature idea. You gather extensive context, present findings, discuss with the user, and clean up the roadmap when a decision is made.

ultrathink

## Phase 1: Input Resolution

1. **Parse $ARGUMENTS** — Determine if this is:
   - **Existing roadmap item:** A reference to a feature in `ROADMAP.md` (or similar roadmap file discovered from CLAUDE.md)
   - **New idea:** A description of something NOT currently in the roadmap

2. **Read the roadmap file** — Search for a matching section. Match by:
   - Exact heading match
   - Keyword overlap with section headings or content
   - If multiple matches, ask user to clarify
   - If no match found, treat as a new idea

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
- **User context relevance** — Use project context from CLAUDE.md (target audience, deployment region, scale, constraints) to evaluate feasibility through the user's actual lens.

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

### Recommendations
[What you recommend — implement as-is, modify approach, defer, drop, or split. Explain why.]

### Open Questions
[Genuine questions for the user that would affect the decision.]
```

**Keep it concise.** Details were gathered for YOUR reasoning. The user gets insights and conclusions.

## Phase 4: Interactive Discussion

After presenting the report, the conversation continues naturally:

- Answer questions with specific evidence from the research
- Explore alternative approaches if the user pushes back
- Do additional targeted research if new angles come up
- Help the user reach a decision

**Do NOT rush to a conclusion.** The discussion ends when the user explicitly indicates a decision:
- "Add it to the backlog" → proceed to action phase
- "Let's plan this" / "make an inline plan" → proceed to action phase
- "Drop it" / "not worth it" → proceed to cleanup phase
- "Modify the roadmap item" → help refine, then wait for next decision
- "Let me think about it" → stop, no cleanup needed

## Phase 5: Action & Cleanup

When the user decides on an action:

### If adding to backlog:

1. Verify Linear MCP: call `mcp__linear__list_teams`. If unavailable, STOP: "Linear MCP not connected. Run `/mcp` to reconnect."
2. Draft Backlog issues following the add-to-backlog patterns (problem-focused descriptions, proper labels and priority)
3. Show proposed issues to user for confirmation before creating
4. After creation, ask: **"Remove this feature from the roadmap file?"**
5. If confirmed → run roadmap cleanup procedure

### If making an inline plan:

1. Summarize what was decided during the discussion
2. Tell the user: "Run `/plan-inline [summary]` to create the implementation plan."
3. Ask: **"Remove this feature from the roadmap file?"**
4. If confirmed → run roadmap cleanup procedure

### If dropping:

1. Ask: **"Remove this feature from the roadmap file?"**
2. If confirmed → run roadmap cleanup procedure

### If modifying:

1. Edit the feature section in the roadmap file with the agreed changes
2. Do NOT remove — the feature stays for future evaluation

### Roadmap cleanup procedure

When removing a feature from the roadmap:
1. Read the roadmap file to get current content
2. Delete the entire feature section (from heading through the separator after it)
3. Remove the feature's row from the Contents table at the top (if one exists)
4. Check remaining features for cross-references to the removed feature — update or remove them
5. Verify file structure is clean (no orphaned separators, no broken links)
6. Follow the roadmap file's conventions section for all modifications (if one exists)

## Rules

- **Evidence over opinions** — Every finding must be backed by specific evidence (code paths, API docs, forum posts, data points)
- **Honest about uncertainty** — If you can't determine something, say so
- **User's context matters** — Use project context from CLAUDE.md (audience, region, scale, constraints) to inform the analysis
- **Don't oversell or undersell** — Present findings neutrally, let the user decide
- **Roadmap conventions** — Follow the roadmap file's conventions section for modifications (if one exists)
- **No implementation** — This skill researches and discusses. It does NOT write code or create implementation plans (except when creating backlog issues as part of the action phase)
- **Concise reports** — Research is thorough, output is scannable

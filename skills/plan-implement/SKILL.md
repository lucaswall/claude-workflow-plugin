---
name: plan-implement
description: Execute the pending plan in PLANS.md using an agent team for parallel implementation. Use when user says "implement the plan", "execute the plan", "team implement", or after any plan-* skill creates a plan. Spawns worker agents that each own distinct file groups to avoid conflicts. Updates Linear issues in real-time. Falls back to single-agent mode if agent teams unavailable.
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__update_issue, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Execute the current pending work in PLANS.md using an agent team for parallel implementation. You are the **team lead/coordinator**. You break the plan into non-overlapping work units, spawn worker agents, coordinate their progress, and handle verification and documentation.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

## Pre-flight Check

1. **Read PLANS.md** — Understand the full context and history
2. **Read CLAUDE.md** — Understand TDD workflow and project rules
3. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
4. **Discover project info from CLAUDE.md** — Read the LINEAR INTEGRATION section to find:
   - Team name (e.g., "Food Scanner")
   - Issue prefix (e.g., PROJ-xxx)
   - If LINEAR INTEGRATION section not found, call `mcp__linear__list_teams` to discover the team
5. **Identify pending work** — Use this priority order:
   - Check latest Iteration block for "Tasks Remaining" section
   - Look for `## Fix Plan` (h2 level) with no iteration after it
   - Original Plan with no "Iteration 1" → Execute from Task 1
   - Nothing pending → Inform user "No pending work in PLANS.md"

## Work Partitioning

This is the critical step. You MUST partition tasks so workers never edit the same files.

### Analyze Task File Ownership

For each pending task in PLANS.md:
1. Read the task's **Files** section to identify all files it creates or modifies
2. Build a map: `task → set of files`
3. Identify any file that appears in more than one task

### Partition Into Work Units

Group tasks into **work units** where:
- Each work unit is assigned to one worker
- **No file appears in more than one work unit** (the hard constraint)
- Tasks within a work unit are ordered by dependency (earlier tasks first)

**Partitioning algorithm:**

1. Build a file-ownership graph: for each file, list which tasks touch it
2. Tasks that share files MUST be in the same work unit (they cannot be parallelized)
3. Tasks with no shared files CAN be in separate work units
4. Group connected components: if Task A shares a file with Task B, and Task B shares a file with Task C, then A, B, and C must all be in the same work unit
5. Merge small work units (1 task) into the nearest related unit to reduce overhead

**Deciding the number of workers:**

| Work units | Workers |
|-----------|---------|
| 1 | 1 (still benefits from dedicated context) |
| 2 | 2 |
| 3 | 3 |
| 4+ | Cap at 4 (diminishing returns, coordination overhead) |

**If all tasks share files** (common for tightly coupled changes):
- Use 1 worker for implementation
- The lead handles coordination and verification only
- This still benefits from the team structure (dedicated implementation context)

### Reserve Generated-File Tasks for the Lead

Some tasks involve running CLI tools that **generate** files (e.g., `npx drizzle-kit generate` for database migrations). These tasks MUST NOT be assigned to workers because:
- Workers tend to hand-write generated files instead of running the command, producing corrupt output
- Generated files (snapshots, lock files, migration metadata) are complex and error-prone to write manually

**How to handle:**
1. During partitioning, identify any task whose steps include running a generator command (e.g., `drizzle-kit generate`, `prisma generate`, `openapi-generator`, etc.)
2. Remove those tasks from worker assignments
3. The lead runs these tasks directly in the Post-Implementation Verification phase, after all workers have completed their source file changes
4. In the partition log, note: "Task N: [title] — reserved for lead (generated files)"

### Verify Partition Correctness

Before spawning workers, verify:
- [ ] Every pending task is assigned to exactly one work unit
- [ ] No file appears in more than one work unit
- [ ] Task ordering within each work unit respects dependencies
- [ ] Each work unit has a clear, non-overlapping scope description
- [ ] No work unit contains tasks that run file-generation CLI tools

**Log the partition plan** — output to the user so they can see how work is divided.

## Team Setup

### Pre-create directories

Before spawning workers, create all directories they will need. This avoids Bash(mkdir) permission prompts that block workers due to known Claude Code bugs.

1. Collect all file paths from every work unit's file ownership list
2. Extract unique parent directories (e.g., `src/lib/__tests__/` from `src/lib/__tests__/session.test.ts`)
3. Run `mkdir -p` for each directory that doesn't already exist

This step is fast and harmless — creating directories that already exist is a no-op.

### Create the team

Use `TeamCreate`:
- `team_name`: "plan-implement"
- `description`: "Parallel plan implementation with file-partitioned workers"

**If TeamCreate fails**, switch to Fallback: Single-Agent Mode (see below).

### Create tasks

Use `TaskCreate` to create one task per work unit:
- Subject: "Work Unit N: [brief description of scope]"
- Description: include the list of plan tasks and files owned

### Spawn workers

Use the `Task` tool with `team_name: "plan-implement"`, `subagent_type: "general-purpose"`, and `model: "sonnet"` for each worker. Give each a `name` like `worker-1`, `worker-2`, etc.

Spawn all workers in parallel (concurrent Task calls in one message).

**IMPORTANT:** Use `mode: "bypassPermissions"` so workers can write files, run tests, and use bash without permission prompts.

### Worker Prompt Template

Each worker gets this prompt (substitute the specific sections):

```
You are an implementation worker for this project. You implement plan tasks following strict TDD workflow.

ASSIGNED TASKS:
{paste the full task descriptions from PLANS.md for this work unit}

FILE OWNERSHIP:
You are responsible for these files ONLY:
{list all files in this work unit}
Do NOT modify any files outside this list.

RULES:
- Follow TDD strictly: write test → run test (expect fail) → implement → run test (expect pass)
- **E2E TEST EXCEPTION:** If your task is writing Playwright E2E tests (files in `e2e/tests/*.spec.ts`), do NOT attempt TDD verification. Write the spec file, but SKIP the "run test" steps entirely. You cannot run E2E tests — they require exclusive access to the build, port, and database. The lead will run all E2E tests after all workers complete. Just write the spec and report completion.
- Never modify files outside your ownership list
- Read CLAUDE.md for project conventions before starting
- Run tests using the verifier agent in TDD mode ONLY: Task tool with subagent_type "verifier" and prompt set to your test file pattern (e.g., "session" or "src/lib/__tests__/session.test.ts"). NEVER run verifier without a test pattern. NEVER run verifier with "e2e". NEVER run `npx playwright` or `npm run e2e` directly. Only the lead runs full verification and E2E tests.
- Report progress to the lead after completing each task
- Report the FINAL summary to the lead when ALL your tasks are done
- Do NOT attempt to update Linear issues — the lead handles all Linear state transitions
- NEVER hand-write generated files (migrations, snapshots, lock files). If a task requires running a CLI generator (e.g., `npx drizzle-kit generate`), report it as a blocker — the lead will handle it.
- NEVER use Bash(mkdir) to create directories. Use the Write tool to create files — it auto-creates parent directories. If you need an empty directory, report it as a blocker and the lead will create it.

WORKFLOW FOR EACH TASK:

For unit/integration test tasks (files in src/):
1. Send progress message to lead: "Starting Task N: [title] [PROJ-XXX]" (include issue ID)
2. Read relevant existing source files to understand patterns
3. Write failing test(s) in the appropriate __tests__/ directory
4. Run tests with verifier in TDD mode (pass your test file pattern as prompt) — confirm test fails
5. Implement the minimal code to make test pass
6. Run tests with verifier in TDD mode (same pattern) — confirm test passes
7. Send progress message to lead: "Completed Task N: [title] [PROJ-XXX]" (include issue ID)
8. Move to next task

For E2E test tasks (files in e2e/tests/*.spec.ts):
1. Send progress message to lead: "Starting Task N: [title] [PROJ-XXX]" (include issue ID)
2. Read existing E2E specs in e2e/tests/ and fixtures in e2e/fixtures/ to understand patterns
3. Write the Playwright spec file — DO NOT attempt to run it
4. Send progress message to lead: "Completed Task N: [title] [PROJ-XXX] — E2E spec written, needs lead verification" (include issue ID)
5. Move to next task

WHEN ALL TASKS ARE DONE:
Send a final summary message to the lead with:
---
WORKER: {worker name}
STATUS: COMPLETE

TASKS COMPLETED:
- Task N: [title] (PROJ-XXX) - [brief description of what was done]

FILES MODIFIED:
- path/to/file.ts - [what changed]
- path/to/test.ts - [what was tested]
---

If you encounter a blocker (dependency on another worker's output, unclear requirements), send a message to the lead describing the blocker. Do NOT guess or work around it.
```

### Assign tasks and label issues

After spawning, for each work unit:
1. Use `TaskUpdate` to assign each task to its worker by name
2. **Label all Linear issues** in the work unit with the worker label using `mcp__linear__update_issue`:
   - Worker 1 issues get label "Worker 1"
   - Worker 2 issues get label "Worker 2"
   - Worker 3 issues get label "Worker 3"
   - Worker 4 issues get label "Worker 4"
   - Add the worker label to the issue's existing labels (do not replace them)

## Linear State Management

**CRITICAL:** Workers do NOT have access to Linear MCP tools. The lead is responsible for ALL Linear state transitions, triggered by worker progress messages.

**When a worker REPORTS starting a task:**
1. Parse the issue ID from the worker's message (e.g., "Starting Task N: [title] [PROJ-XXX]")
2. IMMEDIATELY move the issue to "In Progress" using `mcp__linear__update_issue`

**When a worker REPORTS completing a task:**
1. Parse the issue ID from the worker's message (e.g., "Completed Task N: [title] [PROJ-XXX]")
2. IMMEDIATELY move the issue to "Review" using `mcp__linear__update_issue`
3. Acknowledge the worker's completion

**If a task has no Linear issue link**, skip state updates for that task.

## Coordination (while workers work)

### Message Handling

1. Worker messages are **automatically delivered** to you — do NOT poll
2. Teammates go idle after each turn — this is normal and expected
3. Track progress via `TaskList`
4. As each worker reports task completion, acknowledge receipt
5. If a worker reports a blocker, help resolve it (provide information, adjust file ownership, etc.)

### Handling Blockers

| Blocker Type | Action |
|-------------|--------|
| Worker needs output from another worker's file | Check if the dependency worker has completed that file. If yes, tell blocked worker to re-read it. If no, tell blocked worker to wait and notify when ready. |
| Test failure worker can't resolve | Read the failing test output, provide guidance |
| Unclear requirements | Re-read PLANS.md, provide clarification |
| File conflict detected | STOP the conflicting worker, reassign the file to one worker |

### Progress Tracking

After each worker completion message:
1. **Move completed Linear issues to "Review"** using `mcp__linear__update_issue`
2. Mark completed work units via `TaskUpdate` with `status: "completed"`

## Post-Implementation Verification

Once ALL workers have reported completion:

### 1. Run Lead-Reserved Tasks (Generated Files)

If any tasks were reserved for the lead during partitioning (e.g., Drizzle migration generation):
1. Run the CLI command as specified in PLANS.md (e.g., `npx drizzle-kit generate`)
2. Verify the output files are correct (review generated SQL, snapshots, etc.)
3. If the generator produces no changes or errors, investigate — workers may have missed a schema change

### 2. Run E2E Tests (if workers wrote E2E specs)

If any work unit included writing Playwright E2E test specs (`e2e/tests/*.spec.ts`), run the E2E suite now. Workers write E2E specs without running them — the lead is the only one who can run E2E tests safely.

Run the `verifier` agent in E2E mode:
```
Use Task tool with subagent_type "verifier" and prompt "e2e"
```

If E2E tests fail → fix the spec files directly, then re-run until all pass.

### 3. Run Full Verification

Run the `bug-hunter` agent to review all changes:
```
Use Task tool with subagent_type "bug-hunter"
```

**Bug handling policy — fix ALL real bugs:**
- Every bug reported by bug-hunter must be fixed, whether introduced by this iteration or pre-existing
- "It was already broken before this plan" is NOT a valid reason to skip a bug — fix it anyway
- The ONLY findings you may skip are **verifiable false positives** (the reported behavior is actually correct, the code is not actually buggy)
- If uncertain whether something is a real bug or a false positive, treat it as a real bug and fix it
- Message the relevant worker to fix bugs in their files, or fix directly if workers have shut down

### 4. Run Full Test Suite

Run the `verifier` agent to confirm everything passes together:
```
Use Task tool with subagent_type "verifier"
```

If failures → identify which worker's code is failing, message them to fix, or fix directly.

### 5. Fix Any Integration Issues

Workers implement in isolation. When their code comes together, there may be integration issues:
- Import mismatches
- Type incompatibilities between modules
- Missing shared utilities

Fix these directly (you are allowed to edit files during this phase).

## Document Results

After verification passes, append a new "Iteration N" section to PLANS.md:

```markdown
---

## Iteration N

**Implemented:** YYYY-MM-DD
**Method:** Agent team (N workers)
[OR: **Method:** Single-agent (team unavailable)]

### Tasks Completed This Iteration
- Task 3: Fix session validation - Updated middleware, added tests (worker-1)
- Task 4: Add health check - Created /api/health endpoint (worker-2)

### Tasks Remaining
- Task 5: Add Fitbit token refresh
- Task 6: Add food analysis endpoint
(omit this section if ALL tasks completed)

### Files Modified
- `src/lib/session.ts` - Updated session validation logic
- `src/app/api/health/route.ts` - Created health endpoint

### Linear Updates
- PROJ-9: Todo → In Progress → Review
- PROJ-10: Todo → In Progress → Review

### Pre-commit Verification
- bug-hunter: [Passed | Found N bugs, fixed before proceeding]
- verifier: All N tests pass, zero warnings

### Work Partition
- Worker 1: Tasks 3 (session files)
- Worker 2: Task 4 (health endpoint files)
(omit this section in single-agent mode)

### Continuation Status
[All tasks completed.]
OR
[Point budget reached. More tasks remain.]
```

**IMPORTANT:**
- Do NOT add "Review Findings" or "Notes" sections - reserved for `plan-review-implementation`
- Always list completed tasks in "Tasks Completed This Iteration"
- If stopping early, also list remaining tasks in "Tasks Remaining"
- If ALL tasks are complete, OMIT the "Tasks Remaining" section entirely

**Note:** The presence of "Tasks Remaining" does NOT prevent review. `plan-review-implementation` will review the completed tasks in this iteration regardless.

## Shutdown Team

After documenting results (skip this section in single-agent fallback mode):
1. Send shutdown requests to all workers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations
3. Use `TeamDelete` to remove team resources

## Fallback: Single-Agent Mode

If `TeamCreate` fails (agent teams unavailable), implement the plan sequentially as a single agent:

1. **Inform user:** "Agent teams unavailable. Implementing in single-agent mode."
2. **Follow TDD strictly** for each task:
   - Move Linear issue Todo → In Progress
   - Write failing test → run test (expect fail) → implement → run test (expect pass)
   - Move Linear issue In Progress → Review
3. **Track point budget** as a proxy for context usage:

   | Tool call type | Points |
   |----------------|--------|
   | Glob, Grep, Edit, MCP call (Linear etc.) | 1 |
   | Read, Write | 2 |
   | Bash (test run, build, git) | 3 |
   | Task subagent (verifier, bug-hunter) | 5 |

   | Cumulative points | Action |
   |-------------------|--------|
   | **< 200** | Continue to next task |
   | **200–230** | Continue only if next task is small (≤ 3 files) |
   | **> 230** | **STOP** — run pre-stop checklist immediately |

4. **Pre-stop checklist** (run when stopping, regardless of reason):
   - Run `bug-hunter` agent — fix ALL real bugs found (pre-existing or new; only skip verifiable false positives)
   - Run `verifier` agent — fix any failures or warnings
5. **Document results** — Same Iteration block format (omit Work Partition section)

## Termination: Commit and Push

**MANDATORY:** After team cleanup (or after documenting results in single-agent mode), commit all local changes and push to remote.

**Steps:**
1. Stage modified files: `git status --porcelain=v1`, then `git add <file> ...` for each changed file — **skip** files matching `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
2. Create commit with message format (do **not** include `Co-Authored-By` tags):
   ```
   plan: implement iteration N - [brief summary]

   Tasks completed:
   - Task X: [title]
   - Task Y: [title]

   Method: agent team (N workers)
   ```
   (Use "Method: single-agent" in fallback mode)
3. Push to current branch: `git push`

**Branch handling:**
- If on `main`, create a feature branch first: `git checkout -b feat/[plan-name]`
- If already on a feature branch, push to that branch

## Error Handling

| Situation | Action |
|-----------|--------|
| PLANS.md doesn't exist or is empty | STOP — "No plan found. Run plan-backlog or plan-inline first." |
| PLANS.md has "Status: COMPLETE" (check both header `**Status:** COMPLETE` and bottom `## Status: COMPLETE`) | STOP — "Plan already complete. Create a new plan first." |
| TeamCreate fails | Switch to single-agent fallback mode |
| All tasks share files (1 work unit) | Use 1 worker — still valid, benefits from dedicated context |
| Worker stops without reporting | Send follow-up message. If unresponsive, note as incomplete and fix directly. |
| Integration test failures after merge | Fix directly in the verification phase |
| Worker edits file outside its ownership | Revert the change, re-assign or fix directly |
| Git conflict during commit | Resolve conflicts, re-run verifier |
| Test won't fail in step 2 (single-agent) | Review test logic — ensure it tests new behavior |
| Test won't pass in step 4 (single-agent) | Debug implementation, do not skip |

## Scope Boundaries

**This skill implements plans. It does NOT:**
1. **NEVER create PRs** — PRs are created by plan-review-implementation
2. **NEVER skip failing tests** — Fix them
3. **NEVER modify PLANS.md sections above current iteration** — Append only
4. **NEVER proceed with warnings** — Fix all warnings first
5. **NEVER ask "should I continue?"** — Use context estimation to decide automatically (single-agent mode)

## Rules

- **Partition by file ownership** — The #1 rule. No file in more than one work unit.
- **Follow TDD strictly** — Test before implementation, always
- **Fix ALL real bugs** — Every bug found by bug-hunter must be fixed, whether pre-existing or introduced by this iteration. Only skip findings that are verifiable false positives (not actual bugs). "It existed before" is never a valid excuse to leave a bug unfixed.
- **Fix failures immediately** — Do not proceed with failing tests or warnings
- **Never modify previous sections** — Only append new Iteration section
- **Always commit and push at termination** — Never end without committing progress
- **Document completed AND remaining tasks** — So next iteration knows where to resume
- **Lead updates Linear in real-time** — Workers do NOT have MCP access. Lead moves issues Todo→In Progress when worker reports starting a task, In Progress→Review when worker reports completing it.
- **Cap at 4 workers** — More workers = more overhead, diminishing returns
- **Lead does NOT implement** — Delegate all implementation to workers. Lead only coordinates, verifies, and documents. (Does not apply in single-agent fallback mode.)
- **Lead runs all CLI generators** — Tasks involving `drizzle-kit generate`, `prisma generate`, or similar CLI tools that produce generated files (migrations, snapshots) are reserved for the lead in the post-implementation phase. Workers must never hand-write these files.
- **Workers use TDD-mode verifier only** — Workers must always pass a test pattern to the verifier (e.g., `verifier "session"`). Full verification (unit + lint + build) and E2E tests are exclusive to the lead. Concurrent full/E2E verifier runs corrupt shared resources (`.next/`, port 3001, database).
- **E2E test tasks are write-only for workers** — When a task is writing Playwright E2E specs (`e2e/tests/*.spec.ts`), workers write the spec but do NOT run it. No TDD cycle for E2E specs. The lead runs all E2E tests once during post-implementation verification (step 2). Workers must not run `npx playwright`, `npm run e2e`, or `verifier "e2e"` under any circumstances.
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never stage sensitive files** — Skip `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
- **Log migrations in MIGRATIONS.md** — If any task changes DB schema, renames columns, changes session/token formats, or renames env vars, append a note to `MIGRATIONS.md` describing what changed and what production data is affected. Workers should report migration-relevant changes to the lead; the lead appends to `MIGRATIONS.md` during post-implementation. Do NOT write migration code — only describe the change.

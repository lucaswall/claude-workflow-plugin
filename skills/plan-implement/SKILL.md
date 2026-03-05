---
name: plan-implement
description: Execute the pending plan in PLANS.md using an agent team for parallel implementation. Use when user says "implement the plan", "execute the plan", "team implement", or after any plan-* skill creates a plan. Spawns worker agents in isolated git worktrees for full code isolation. Updates Linear issues in real-time. Falls back to single-agent mode if agent teams unavailable.
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__update_issue, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Execute the current pending work in PLANS.md using an agent team for parallel implementation. You are the **team lead/coordinator**. You break the plan into domain-based work units, create isolated git worktrees for each worker, spawn worker agents, coordinate their progress, merge their work, and handle verification and documentation.

Each worker operates in its own **git worktree** — a fully isolated working directory with its own branch, staging area, and `node_modules`. Workers cannot corrupt each other's files. Task assignment is **domain-based** — overlapping file edits are acceptable and resolved by the lead during the merge phase.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

## Pre-flight Check

1. **Read PLANS.md** — Understand the full context and history
2. **Read CLAUDE.md** — Understand TDD workflow and project rules
3. **Verify Linear MCP** — Call `mcp__linear__list_issues` with `team: "<discovered-team-name>"` and `state: "Todo"` **directly** (never delegate to a subagent — subagents don't have MCP access). If the tool is unavailable or errors, **STOP immediately** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill." Do NOT rationalize continuing without Linear.
4. **Identify pending work** — Use this priority order:
   - Check latest Iteration block for "Tasks Remaining" section
   - Look for `## Fix Plan` (h2 level) with no iteration after it
   - Original Plan with no "Iteration 1" → Execute from Task 1
   - Nothing pending → Inform user "No pending work in PLANS.md"

## Scope Assessment

Before partitioning into work units, assess whether workers are justified. Worker overhead (worktree setup, team creation, spawning, coordination, merge, cleanup) is significant — only use workers when the implementation work clearly exceeds that overhead.

### Step 1: Classify each task

For each pending task/fix, estimate its size:

| Size | Description | Examples |
|------|-------------|---------|
| **0** | No TypeScript code — docs, config, skill files only | Update CLAUDE.md, edit .env.example, modify SKILL.md |
| **S** | Single-line or few-line surgical change | Replace one API call, add try/catch, add an attribute, fix a condition |
| **M** | Moderate implementation with tests | New error state with tests, add timeout+logging to multiple calls |
| **L** | Substantial new code or multi-file feature | New component, new API route, refactor a module, implement a protocol |

### Step 2: Compute effective scope

1. **Count independent work units** — Group tasks that share files into a single unit. E.g., two fixes both touching `claude.ts` = 1 unit, not 2.
2. **Estimate total effort** — Sum the task sizes: 0=0, S=1, M=2, L=4.

### Step 3: Decision

| Independent work units | Total effort | Decision |
|------------------------|-------------|----------|
| 1 unit (any effort) | Any | **Single-agent** — no parallelism benefit |
| 2+ units | ≤6 points | **Single-agent** — worker overhead exceeds implementation time |
| 2+ units | 7–11 points | **Workers if ≥3 units**, otherwise single-agent |
| 2+ units | ≥12 points | **Workers** — parallelism pays off |

Announce the decision with reasoning: "N tasks across M independent units, effort score P — [workers/single-agent mode]." Then jump to "Fallback: Single-Agent Mode" if single-agent, or continue to "Work Partitioning" if workers.

**Rationale:** Pure task/file counts miss complexity. Five surgical fixes (5×S=5 points) don't justify workers even across 7 files, but four substantial features (4×L=16 points) clearly do. The effort score captures this. Calibrated from real iterations: a batch of 7 mixed M/L tasks across 3 domains → workers justified and succeeded; a batch of 3 small fixes → worker overhead wasted more time than the fixes took; a fix plan of 5 S-sized tasks → single-agent was correct.

## Work Partitioning

Group pending tasks into work units by **domain** — related areas of the codebase that form coherent implementation units. Workers MAY touch overlapping files; the lead resolves conflicts during the merge phase.

### Analyze Task Domains

For each pending task in PLANS.md:
1. Read the task description and files to understand its domain
2. Identify the primary layer: types/schema → service/business logic → API routes → UI components
3. Note cross-cutting concerns (shared types, utilities, config)

### Partition Into Work Units

Group tasks into work units where:
- Tasks in the same domain or tightly coupled belong together
- Cross-cutting tasks go with the domain they most closely relate to
- Work is spread roughly evenly across units

**Partitioning guidelines:**
1. Group by functional domain: "auth flow," "data pipeline," "notification system"
2. Prefer grouping tasks that depend on each other's output
3. When in doubt, group tasks together rather than splitting them

**Deciding the number of workers:**

| Work units | Workers |
|-----------|---------|
| 1 | 1 (still benefits from dedicated context) |
| 2 | 2 |
| 3 | 3 |
| 4+ | Cap at 4 (diminishing returns, coordination overhead) |

### Reserve Generated-File Tasks for the Lead

Tasks involving CLI tools that **generate** files (e.g., `npx drizzle-kit generate`) MUST NOT be assigned to workers. Workers hand-write generated files instead of running the command, producing corrupt output.

**How to handle:**
1. Identify any task whose steps include a generator command
2. Remove those tasks from worker assignments
3. The lead runs them after the merge phase
4. Note in partition log: "Task N: [title] — reserved for lead (generated files)"

### Verify Partition

Before proceeding, verify:
- [ ] Every pending task is assigned to exactly one work unit (or reserved for lead)
- [ ] Task ordering within each work unit respects dependencies
- [ ] Each work unit has a clear scope description
- [ ] No work unit contains tasks that run file-generation CLI tools

**Log the partition plan** — output to the user so they can see how work is divided.

## Worktree Setup

### Determine Feature Branch

If on `main`, create a feature branch:
```bash
git checkout -b feat/<plan-name>
```
If already on a feature branch, stay on it. Record the branch name as `FEATURE_BRANCH`.

### Clean Up Previous Runs

Remove any leftover worktrees and branches from a previous failed run:
```bash
git worktree prune
# For each worker N:
git branch -D <FEATURE_BRANCH>-worker-N 2>/dev/null || true
rm -rf _workers/
```

### Create Worker Worktrees

For each worker:
```bash
git worktree add _workers/worker-N -b <FEATURE_BRANCH>-worker-N
```

**IMPORTANT:** Use a hyphen (`-worker-N`), NOT a slash (`/worker-N`). Git cannot create `refs/heads/feat/foo-123/worker-1` when `refs/heads/feat/foo-123` already exists as a branch ref.

Example: if `FEATURE_BRANCH` is `feat/proj-123-notifications`, worker branches are:
- `feat/proj-123-notifications-worker-1`
- `feat/proj-123-notifications-worker-2`

### Bootstrap Worktree Environments

**Pre-check:** Verify `.gitignore` covers symlinks before creating them. The `node_modules/` entry (with trailing slash) only matches directories — a symlink is a file and won't be excluded. Ensure a bare `node_modules` entry exists:
```bash
grep -q '^node_modules$' .gitignore || sed -i '' '/^node_modules\//i\
node_modules' .gitignore
```

Each worktree needs dependencies and environment variables:
```bash
# For each worker N:
ln -s "$(pwd)/node_modules" _workers/worker-N/node_modules
cp .env _workers/worker-N/.env 2>/dev/null || true
cp .env.local _workers/worker-N/.env.local 2>/dev/null || true
```

**Why symlink, not copy:** `cp -r node_modules` breaks `.bin/` symlinks on macOS — `cp -r` dereferences symlinks, turning `.bin/vitest -> ../vitest/vitest.mjs` into a regular file containing `import './dist/cli.js'` that can't resolve. Symlinking is instant and avoids the issue entirely. Workers don't install packages, so a shared read-only reference is safe.

### Worktree Setup Failure

If `git worktree add` fails:
1. Clean up: `git worktree prune && rm -rf _workers/`
2. Delete any created branches: `git branch -D <FEATURE_BRANCH>-worker-N 2>/dev/null || true`
3. Fall back to single-agent mode
4. Inform user: "Worktree setup failed. Falling back to single-agent mode."

## Team Setup

### Create the team

Use `TeamCreate`:
- `team_name`: "plan-implement"
- `description`: "Parallel plan implementation with worktree-isolated workers"

**If TeamCreate fails**, clean up worktrees and switch to Fallback: Single-Agent Mode.

### Create tasks

Use `TaskCreate` for each work unit:
- Subject: "Work Unit N: [brief scope/domain description]"
- Description: list of plan tasks assigned to this unit

### Spawn workers

Use `Task` tool with `team_name: "plan-implement"`, `subagent_type: "general-purpose"`, `model: "sonnet"`, and `mode: "bypassPermissions"` for each worker. Name them `worker-1`, `worker-2`, etc.

Spawn all workers in parallel (concurrent Task calls in one message).

### Worker Prompt Template

Read [references/worker-prompt-template.md](references/worker-prompt-template.md) for the full worker prompt template, testing context examples, and protocol consistency block.

### Assign tasks and label issues

After spawning, for each work unit:
1. `TaskUpdate` to assign each task to its worker by name
2. Label Linear issues with worker label using `mcp__linear__update_issue`:
   - Worker 1 → "Worker 1", Worker 2 → "Worker 2", etc.
   - Add label to existing labels (don't replace)

## Linear State Management

**CRITICAL:** Workers do NOT have access to Linear MCP tools. The lead handles ALL Linear state transitions.

**When a worker REPORTS starting a task:**
1. Parse the issue ID from the worker's message
2. IMMEDIATELY move the issue to "In Progress" using `mcp__linear__update_issue`

**When a worker REPORTS completing a task:**
1. Parse the issue ID from the worker's message
2. IMMEDIATELY move the issue to "Review" using `mcp__linear__update_issue`
3. Acknowledge the worker's completion

**If a task has no Linear issue link**, skip state updates for that task.

## Coordination (while workers work)

### Worker Startup Grace Period

After spawning workers, wait at least **5 minutes** before taking any corrective action. Workers need 2–4 turns to: cd to workspace, validate environment, read CLAUDE.md, read source files, and send their first "Starting Task" message.

**During the grace period:**
- Idle notifications are EXPECTED and normal — do not react to them
- Do NOT send status check messages until 5 minutes have passed
- Do NOT delete worktrees, remove branches, or clean up
- You MAY acknowledge worker messages if they arrive

**After 5 minutes with no messages from a worker:**
1. Check the worktree for file modifications: `git -C _workers/worker-N status --short`
2. If files are modified → worker IS making progress silently. Wait 3 more minutes.
3. If NO files modified → send ONE status check message. Wait 2 more minutes.
4. If still no response and no file changes → the worker is stuck. Do NOT delete its worktree. Instead, fall back to single-agent mode for that worker's tasks (implement them yourself in the main workspace). Leave the worktree intact until the post-worker cleanup phase.

### Lead Non-Interference Rule

While workers are actively working (uncommitted changes visible in their worktree):
- Do NOT read or debug their source files from the main workspace
- Do NOT attempt to fix their tests or implementation
- DO check their worktree status to confirm activity: `git -C _workers/worker-N status --short`
- Only intervene if: (a) worker explicitly reports a blocker via message, OR (b) worker is idle with no file changes for 5+ minutes after the grace period

If the user reports a worker is struggling, check worktree status first. If changes exist, the worker is making progress — report this to the user and wait. Workers often hit temporary test failures and self-resolve within a few turns.

### Message Handling

1. Worker messages are **automatically delivered** — do NOT poll
2. Teammates go idle after each turn — normal and expected
3. Track progress via `TaskList`
4. When a worker reports ALL tasks complete and has committed:
   a. Acknowledge completion
   b. Update Linear issues (In Progress → Review)
   c. Mark their TaskList task as completed via `TaskUpdate`
   d. **Immediately send shutdown request** via `SendMessage` with `type: "shutdown_request"` — do not wait for other workers to finish
5. When the **last worker confirms shutdown**, call `TeamDelete` immediately — the team is no longer needed. Deleting it now prevents bug-hunter/verifier subagents from accidentally joining as team members.
6. If a worker reports a blocker, help resolve it

### Handling Blockers

| Blocker Type | Action |
|-------------|--------|
| Worker needs another worker's code | Tell them to proceed with their best assumption — conflicts resolved at merge |
| Test failure worker can't resolve | Read the failing test output, provide guidance |
| Unclear requirements | Re-read PLANS.md, provide clarification |
| Generated file needed | Acknowledge — lead handles it post-merge |

## Post-Worker Phase

Once ALL workers have reported completion and committed their changes:

### 1. Pre-Shutdown Verification

Before sending any shutdown requests, verify each worker's state:
```bash
# For each worker N:
git -C _workers/worker-N log --oneline -1
git -C _workers/worker-N status --short
```

If a worker has uncommitted changes (files listed by `status --short`), salvage them:
```bash
git -C _workers/worker-N add -A -- ':!node_modules' ':!.env' ':!.env.local'
git -C _workers/worker-N commit -m "lead: salvage worker-N uncommitted progress"
```

### 2. Verify All Workers Shut Down and Team Deleted

Workers should already be shut down individually during the Coordination phase (each shut down as they completed). Verify:
- All work unit tasks are marked completed via `TaskList`
- `TeamDelete` was called after the last shutdown confirmation

**If any worker was NOT shut down during coordination** (e.g., went idle without reporting), salvage their work (step 1) and send shutdown now. Call `TeamDelete` after the last confirmation.

**CRITICAL: Never delete worktrees while workers are alive.** Worktree deletion is IRREVERSIBLE and destroys all uncommitted worker progress. The sequence MUST be: shutdown all workers → verify all confirmed → THEN delete worktrees in the Cleanup phase.

### 3. Merge Worker Branches

Merge worker branches into the feature branch **one at a time, foundation-first**.

**Determine merge order:**
- Workers handling lower-level code merge first: types/schemas → services → API routes → UI
- If workers are at the same layer, merge by worker number
- The first merge is always a fast-forward (feature branch hasn't moved)

**For each worker branch (in order):**
```bash
git merge <FEATURE_BRANCH>-worker-N
```

**After each merge (starting from the second):**
```bash
npm run typecheck
```
If type errors → fix them before merging the next worker. This catches integration issues early before they compound.

**If a merge has conflicts:**
1. Review the conflicting files — understand both workers' intent from the plan
2. Resolve conflicts, keeping correct logic from both sides
3. **Verify no conflict markers remain:** `grep -rn '<<<<<<\|======\|>>>>>>' <resolved-files>` — fix any stray markers before committing
4. `git add` resolved files, then `git commit` (git's auto-generated merge message is fine)
5. Run `npm run typecheck` before continuing to the next merge

**If `git merge` fails entirely** (e.g., worktree artifacts like committed symlinks):
1. Fall back to cherry-pick: `git cherry-pick <FEATURE_BRANCH>-worker-N --no-commit`
2. Unstage any worktree artifacts: `git reset HEAD node_modules 2>/dev/null`
3. Commit: `git commit -m "fix: [worker summary]"`
4. Verify `node_modules` is still a real directory (not a symlink): `ls -ld node_modules | head -1`
5. If it became a symlink: `rm -f node_modules && npm install`

### 4. Run Lead-Reserved Tasks (Generated Files)

If any tasks were reserved for the lead during partitioning:
1. Run the CLI command (e.g., `npx drizzle-kit generate`)
2. Verify output files are correct
3. If the generator produces no changes, investigate — workers may have missed a schema change

### 5. Install New Dependencies (if needed)

If the plan required new npm packages that workers couldn't install:
```bash
npm install <package-name>
```

### 6. Run Post-Merge Integration Tests

Run the full unit/integration test suite immediately after all merges:
```bash
npm test
```

**Why here (before bug-hunter):** Workers only run targeted tests (`npx vitest run "pattern"`) in their worktrees. Cross-domain integration bugs (missing events on certain paths, stale closures at boundaries, type mismatches between worker outputs) only surface when all code is merged and the full suite runs. Catching these before bug-hunter reduces the bug-hunter's job to logic issues that tests don't cover.

If failures → fix directly, then re-run until all tests pass.

### 7. Run E2E Tests (if workers wrote E2E specs)

Run the `verifier` agent in E2E mode:
```
Task tool with subagent_type "verifier" and prompt "e2e"
```
If E2E tests fail → fix the specs directly, then re-run.

### 8. Run Full Verification

**Bug hunter:**
```
Task tool with subagent_type "bug-hunter"
```

Fix ALL real bugs — pre-existing or new. Only skip verifiable false positives.

**Verifier (tests + lint + build):**
```
Task tool with subagent_type "verifier"
```

If failures → fix directly (workers are shut down by this point).

## Document Results

After verification passes, append a new "Iteration N" section to PLANS.md using the template in [references/iteration-template.md](references/iteration-template.md).

## Cleanup

After documenting results (skip in single-agent fallback mode), the lead MUST clean up worktrees, worker branches, sync dependencies, and verify clean state. Follow [references/cleanup-procedures.md](references/cleanup-procedures.md) for the full step-by-step procedure.

## Fallback: Single-Agent Mode

If `TeamCreate` fails or worktree setup fails, implement the plan sequentially as a single agent:

1. **Inform user:** "Agent teams/worktrees unavailable. Implementing in single-agent mode."
2. **Clean up** any partially created worktrees: `git worktree prune && rm -rf _workers/`
3. **Follow TDD strictly** for each task:
   - Move Linear issue Todo → In Progress
   - Write failing test → run test (expect fail) → implement → run test (expect pass)
   - Move Linear issue In Progress → Review
4. **Track point budget** as a proxy for context usage:

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

5. **Pre-stop checklist** (run when stopping, regardless of reason):
   - Run `bug-hunter` agent — fix ALL real bugs found
   - Run `verifier` agent — fix any failures or warnings
6. **Document results** — Same Iteration block format (omit Work Partition and Merge Summary)

## Termination: Commit and Push

**MANDATORY:** After cleanup (or after documenting results in single-agent mode), commit all changes and push.

**Steps:**
1. Stage modified files: `git status --porcelain=v1`, then `git add <file> ...` — **skip** files matching `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
2. Create commit with a **simple `-m` string** (do **not** include `Co-Authored-By` tags):
   ```bash
   git commit -m "plan: implement iteration N - [brief summary]"
   ```
   Use "Method: single-agent" in fallback mode. Keep the message on one line -- task details are already in PLANS.md.
   **IMPORTANT:** Do NOT use `git commit -m "$(cat <<'EOF'...)"` or any `$()` subshell -- the subshell triggers permission prompts even with `Bash(git *)` in the allow list.
3. Push to current branch: `git push`

**Branch handling:**
- If on `main`, create a feature branch first: `git checkout -b feat/[plan-name]`
- If already on a feature branch, push to that branch

## Error Handling

| Situation | Action |
|-----------|--------|
| PLANS.md doesn't exist or is empty | STOP — "No plan found. Run plan-backlog or plan-inline first." |
| PLANS.md has "Status: COMPLETE" | STOP — "Plan already complete. Create a new plan first." |
| `git worktree add` fails | Clean up, fall back to single-agent mode |
| TeamCreate fails | Clean up worktrees, switch to single-agent fallback |
| Worker branch already exists | Delete it first: `git branch -D <branch> 2>/dev/null` |
| All tasks in same domain (1 unit) | Use 1 worker — still benefits from isolated context |
| Worker stops without reporting | Check worktree: `git -C _workers/worker-N status --short`. If changes exist, salvage and commit from lead. If empty, implement tasks in single-agent mode. Do NOT delete the worktree until shutdown is confirmed. |
| Worker reports workspace missing | Worktree was deleted prematurely. Shut down the worker. Implement its tasks in single-agent mode. |
| Worker's Bash environment breaks | Known bug (#17321) — worker used Bash for file ops. Shut down the worker. Implement its tasks in single-agent mode. |
| Small batch (low effort score) | Skip workers entirely — use single-agent mode from the start (see Scope Assessment) |
| Merge conflict | Resolve in feature branch, run typecheck, continue merging |
| Type errors after merge | Fix before merging next worker |
| Integration failures after all merges | Fix directly in verification phase |
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

- **Domain-based partitioning** — Group tasks by functional domain. Overlapping files are acceptable; the lead resolves conflicts at merge time.
- **Follow TDD strictly** — Test before implementation, always
- **Fix ALL real bugs** — Every bug found by bug-hunter must be fixed, whether pre-existing or new. Only skip verifiable false positives.
- **Fix failures immediately** — Do not proceed with failing tests or warnings
- **Never modify previous sections** — Only append new Iteration section to PLANS.md
- **Always commit and push at termination** — Never end without committing progress
- **Document completed AND remaining tasks** — So next iteration knows where to resume
- **Lead updates Linear in real-time** — Workers do NOT have MCP access
- **Cap at 4 workers** — More = more overhead, diminishing returns
- **Lead does NOT implement** — Delegate all implementation to workers. Lead only coordinates, merges, verifies, and documents. (Exception: single-agent fallback and post-merge fixes.)
- **Lead runs all CLI generators** — Drizzle-kit, prisma generate, etc. reserved for lead post-merge
- **Workers test via vitest only** — `npx vitest run "pattern"` in their worktree. No build, no full suite, no E2E.
- **E2E test tasks are write-only for workers** — Workers write specs but do NOT run them
- **Foundation-first merge order** — Merge lower-level workers first (types → services → routes → UI). Typecheck gate (`npm run typecheck`) after each merge. After resolving conflicts, always verify no stray `<<<<<<` markers remain.
- **Workers commit, don't push** — Workers `git add -A && git commit` in their worktree. Lead merges locally via the shared git object database.
- **Docs tasks need explicit values** — When a docs-only worker must document another worker's implementation details (column names, status values, etc.), include the exact values in the worker's prompt. Don't rely on the worker inferring them from the plan.
- **Never delete worktrees while workers are alive** — Worktree deletion is irreversible. Always shutdown workers first, then verify shutdown, then delete worktrees.
- **Respect the 5-minute grace period** — Workers need multiple turns to start. Do not send status checks or take corrective action before 5 minutes have passed.
- **Small batches skip workers** — Use the effort-point scoring (0=0, S=1, M=2, L=4) to decide. Single-agent when: 1 work unit, ≤6 total points, or 7–11 points with <3 units. Docs-only tasks (CLAUDE.md, .env, skill files) score 0 — they don't justify worker overhead.
- **Always clean up worktrees** — Remove worktrees, prune metadata, delete worker branches after merge
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never stage sensitive files** — Skip `.env*`, `*.key`, `*.pem`, `credentials*`, `secrets*`
- **Log migrations in MIGRATIONS.md** — Workers report migration-relevant changes to lead; lead appends to MIGRATIONS.md. All schema/folder changes MUST include migration logic (startup detection + automatic migration) — never ship a breaking change to persistent data without a migration path.

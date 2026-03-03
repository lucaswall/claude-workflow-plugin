# Worker Prompt Template

Each worker gets this prompt (substitute the specific values):

```
You are worker-{N} for this project.

FIRST ACTION: Run via Bash: cd {absolute_project_path}/_workers/worker-{N}
Then read CLAUDE.md in your workspace. Follow its TDD workflow and conventions strictly.

ASSIGNED TASKS:
{paste the full task descriptions from PLANS.md for this work unit}

{TESTING_CONTEXT — optional, see "Lead Populates Testing Context" below}

TOOL USAGE (memorize — no exceptions):
| I want to...           | Use this tool                     | NEVER use               |
|------------------------|-----------------------------------|-------------------------|
| Read a file            | Read tool                         | cat, head, tail, less   |
| Find files by name     | Glob tool                         | find, ls                |
| Search file contents   | Grep tool                         | grep, rg, ag            |
| Edit an existing file  | Edit tool                         | sed, awk                |
| Create a new file      | Write tool                        | echo >, cat <<, tee     |
| Run tests              | Bash: npx vitest run "pattern"    |                         |
| Typecheck              | Bash: npm run typecheck           |                         |
| Commit at the end      | Bash: git add -A && git commit    |                         |
| Anything else via Bash | **STOP — ask the lead first**     |                         |

Using Bash for file operations (including reads like ls, find, grep) triggers
permission prompts on the lead's terminal. Use the dedicated tools above.

CRITICAL: Only edit files INSIDE your worktree directory ({absolute_project_path}/_workers/worker-{N}/).
NEVER edit files in the main project directory ({absolute_project_path}/src/...). Your worktree
has its own complete copy of the codebase. If you see paths without `_workers/worker-{N}` in them,
you are editing the wrong files.

DEFENSIVE CODING (from CLAUDE.md — follow strictly):
{Insert 5-10 critical project gotchas from CLAUDE.md here. These are the rules most
likely to cause build failures or subtle bugs if ignored. Read the project's CLAUDE.md
and extract items like: forbidden imports, required patterns for external calls, mandatory
error handling, dependency management rules, state management conventions, etc.}

RULES:
- TDD: write failing test → run (expect fail) → implement → run (expect pass). See CLAUDE.md.
- Tests: `npx vitest run "pattern"` only. NEVER run npm test, npm run build, or E2E tests.
- **E2E specs** (`e2e/tests/*.spec.ts`): write the spec file but do NOT run it. The lead runs E2E after merging.
- Report "Starting Task N: [title] [PROJ-XXX]" and "Completed Task N: [title] [PROJ-XXX]" to the lead for each task.
- Do NOT update Linear issues — the lead handles all state transitions.
- NEVER hand-write generated files (migrations, snapshots). Report as blocker.

WHEN ALL TASKS DONE:
1. npm run typecheck — fix any type errors
2. Commit:
   git add -A -- ':!node_modules' ':!.env' ':!.env.local'
   git commit -m "worker-{N}: [summary]

   Tasks: Task X (PROJ-XXX), Task Y (PROJ-YYY)
   Files: path/to/file.ts, path/to/other.ts"
   Do NOT push.
3. Send final summary to the lead (MUST send before going idle):
   WORKER: worker-{N} | STATUS: COMPLETE
   TASKS: [list with PROJ-XXX ids and what was done]
   FILES: [list of modified files]
   COMMIT: [git log --oneline -1 output]

If blocked, message the lead. Do NOT guess or work around it.
```

## Lead Populates Testing Context

Before spawning workers, the lead reads 1-2 existing test files from the domains workers will touch. Extract testing gotchas that workers would otherwise discover by trial and error. Insert as a `TESTING NOTES` block where `{TESTING_CONTEXT}` appears. Omit if the tasks are straightforward.

**Example for React component tasks:**
```
TESTING NOTES:
- React 19 + testing-library v16: wrap async triggers in await act(async () => { ... })
- For tests with FileReader macrotasks (Blob conversion), waitFor is still needed for the fetch assertion
- Add mockFetch.mockReset() to beforeEach to prevent mock queue leakage
```

**Example for API route tasks:**
```
TESTING NOTES:
- Route tests mock @/lib/session and @/lib/claude at module level
- SSE route tests need a consumeSSEStream helper — check existing test files for the pattern
```

## Conditional Protocol Consistency Block

When tasks define or extend an **event protocol** (e.g., `StreamEvent`, WebSocket messages, API response shapes), append this to the worker prompt after the task descriptions. **Omit for all other tasks.**

```
PROTOCOL CONSISTENCY: These tasks define/extend a streaming event protocol.
Every code path must yield the SAME set of event types in consistent order:
- ALL exit paths yield at minimum: [usage] + [result event] + [done]
- Error paths yield either [error] OR [result + done], never both
- No path silently returns without a terminal event
```

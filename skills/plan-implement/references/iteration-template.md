# Iteration Documentation Template

After verification passes, append a new "Iteration N" section to PLANS.md:

```markdown
---

## Iteration N

**Implemented:** YYYY-MM-DD
**Method:** Agent team (N workers, worktree-isolated)
[OR: **Method:** Single-agent (team unavailable)]

### Tasks Completed This Iteration
- Task 3: Fix session validation - Updated middleware, added tests (worker-1)
- Task 4: Add health check - Created /api/health endpoint (worker-2)

### Tasks Remaining
- Task 5: Add token refresh
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
- Worker 1: Tasks 3 (auth domain — session, middleware)
- Worker 2: Task 4 (API domain — health endpoint)
(omit in single-agent mode)

### Merge Summary
- Worker 1: fast-forward (no conflicts)
- Worker 2: merged, 1 conflict in src/types/index.ts (resolved)
(omit in single-agent mode)

### Continuation Status
[All tasks completed.]
OR
[Point budget reached. More tasks remain.]
```

## Important Notes

- Do NOT add "Review Findings" or "Notes" sections - reserved for `plan-review-implementation`
- Always list completed tasks in "Tasks Completed This Iteration"
- If stopping early, also list remaining tasks in "Tasks Remaining"
- If ALL tasks are complete, OMIT the "Tasks Remaining" section entirely
- The presence of "Tasks Remaining" does NOT prevent review. `plan-review-implementation` will review the completed tasks regardless.

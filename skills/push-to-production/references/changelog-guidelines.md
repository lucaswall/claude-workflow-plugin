# Changelog Guidelines

This is a **product changelog** — every entry must describe something a user of the app would notice or care about. Think "what changed when I open the app?" not "what code was written."

## INCLUDE

- New features or screens users interact with
- Changes to existing user-visible behavior or UI
- Bug fixes that affected users
- Performance improvements users would notice

## Key Principle: Net Effect from Production

The changelog describes the **net difference between current production and the new release** — NOT a commit-by-commit replay of the staging cycle. Always compare against `origin/release` (production) when deciding what to include.

**Staging-internal churn gets zero entries.** Examples:

- Bug introduced in staging commit A, fixed in staging commit B → neither appears (production never had the bug)
- Feature implemented, then reworked or redesigned before release → one entry describing the final version, not the journey
- Code added then removed within the same cycle → zero entries
- Fix for a regression that only existed in staging → zero entries

When in doubt, ask: "Would a user on the current production version notice this change?" If not, skip it.

## EXCLUDE — never add entries for

- Staging-internal fixes (bugs that only existed in staging, never in production)
- Changes that cancel each other out within the release cycle
- Rework/iteration on features introduced in the same cycle (only describe the final result)
- Internal API endpoints (these are implementation details behind user-facing features)
- Internal component/hook/utility names (describe what the user sees, not `FooComponent`)
- Skill, tooling, or Claude Code changes
- Infrastructure changes (deployment, env vars, internal architecture)
- Internal implementation details (metadata storage, data cleanup jobs, defensive checks, railguards)
- Linear issue numbers (e.g., PROJ-224) — meaningless to users

## Writing Style

- Describe from the user's perspective: "Dashboard now shows weekly progress" not "Added WeeklyChart component"
- Never expose component names, hook names, or route paths
- One commit can map to zero entries (if purely internal) or one entry
- Multiple commits can be grouped into a single entry

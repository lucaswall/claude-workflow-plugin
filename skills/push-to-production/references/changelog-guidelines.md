# Changelog Guidelines

This is a **product changelog** — every entry must describe something a user of the app would notice or care about. Think "what changed when I open the app?" not "what code was written."

## INCLUDE

- New features or screens users interact with
- Changes to existing user-visible behavior or UI
- Bug fixes that affected users
- Performance improvements users would notice

## EXCLUDE — never add entries for

- Internal API endpoints (these are implementation details behind user-facing features)
- Internal component/hook/utility names (describe what the user sees, not `FooComponent`)
- Skill, tooling, or Claude Code changes
- Infrastructure changes (deployment, env vars, internal architecture)
- Internal implementation details (metadata storage, data cleanup jobs, defensive checks, railguards)
- Linear issue numbers (e.g., FOO-224) — meaningless to users

## Writing Style

- Describe from the user's perspective: "Calorie ring now shows calories burned" not "Added CalorieRing budget marker component"
- Never expose component names, hook names, or route paths
- One commit can map to zero entries (if purely internal) or one entry
- Multiple commits can be grouped into a single entry

---
name: push-to-production
description: Promote main to release with production DB backup and migration handling. Use when user says "push to production", "release", "deploy to production", or "promote to release". Backs up production DB, assesses MIGRATIONS.md, writes migration code if needed, and merges main to release.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, Task, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__update_issue, mcp__linear__list_issue_statuses
argument-hint: [version]
disable-model-invocation: true
---

Promote `main` to `release` with automated backup, migration assessment, and merge. This is the only path to production.

**IMPORTANT:** This skill assumes Drizzle ORM for database migrations. For other ORMs, the migration internals section needs adaptation.

## Phase 1: Pre-flight Checks

### 1.1 Verify Linear MCP

Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

Extract the team name from the response (there should only be one team in most cases).

### 1.2 Git State

```bash
git branch --show-current
git status --porcelain
```

**Requirements:**
- Must be on `main` branch
- Working tree must be clean (no uncommitted changes)
- Must be up to date with remote: `git fetch origin && git rev-list --count HEAD..origin/main` must be `0`

If any check fails, **STOP** and tell the user what to fix.

### 1.3 Docker (for migration validation)

Ensure Docker is available for migration validation:

```bash
docker --version
```

- If available: proceed
- If not: **STOP** — "Docker is not installed. Install Docker to enable migration validation."

Then ensure local Postgres is running (consult CLAUDE.md for docker compose file location):

```bash
docker compose ps
```

If the database service is not running, start it: `docker compose up -d`

### 1.4 Verify Drizzle Migration Internals

Only needed if MIGRATIONS.md has entries (checked in Phase 2, but verify early to fail fast). Read the Drizzle migrator source from the exact version in `node_modules` and confirm our assumptions about the journal table still hold:

```bash
# Check migrator comparison logic
grep -q '"drizzle"."__drizzle_migrations"' node_modules/drizzle-orm/pg-proxy/migrator.js
grep -q 'ORDER BY created_at DESC LIMIT 1' node_modules/drizzle-orm/pg-proxy/migrator.js
grep -q 'created_at' node_modules/drizzle-orm/pg-proxy/migrator.js

# Check hash algorithm
grep -q 'createHash("sha256")' node_modules/drizzle-orm/migrator.js
```

All four must match. If any fails, **STOP**: "Drizzle migration internals have changed since this skill was written. The journal insert approach may not work. Check `node_modules/drizzle-orm/pg-proxy/migrator.js` manually and update the skill."

This verification is safe because we deploy the same `node_modules` — the code we check here is the code that will run in production.

### 1.5 Build & Tests

Run the `verifier` agent (full mode) to confirm unit tests, lint, and build pass:

```
Use Task tool with subagent_type "verifier"
```

If verifier reports failures, **STOP**. Do not proceed with a broken build.

### 1.6 E2E Tests

Run the `verifier` agent in E2E mode to confirm end-to-end tests pass:

```
Use Task tool with subagent_type "verifier" with prompt "e2e"
```

Docker is already verified in Phase 1.3, so prerequisites are met.

If E2E tests fail, **STOP**. Do not proceed — E2E failures indicate integration issues that must be fixed before release.

### 1.7 Release Branch Exists

```bash
git rev-parse --verify origin/release
```

If `release` branch doesn't exist, **STOP** and tell the user to create it.

### 1.8 Diff Assessment

Check what's changing between `release` and `main`:

```bash
git log origin/release..origin/main --oneline
git diff origin/release..origin/main --stat
```

If there are no commits to promote, **STOP**: "Nothing to promote. `main` and `release` are identical."

Show the user the commit list and file diff summary.

## Phase 2: Assess Migrations

Migration assessment runs before backup so we know whether to stop the service first.

### 2.1 Read MIGRATIONS.md

Read `MIGRATIONS.md` from project root.

If `MIGRATIONS.md` has no entries (only the template header), skip to Phase 4 (backup without stopping service, then merge).

### 2.2 Analyze Changes

For each entry in MIGRATIONS.md:

1. **Read the referenced files** on `main` to understand the current (new) state
2. **Compare with `release`** to understand what production currently has:
   ```bash
   git diff origin/release..HEAD -- <relevant-files>
   ```
3. **Determine the net migration** — what SQL or data transformation is needed to go from production's current state to the new state

### 2.3 Classify Migration Complexity

Classify **each MIGRATIONS.md entry independently**. A single release may have entries at different levels.

| Complexity | Criteria | Action |
|-----------|---------|--------|
| **None** | No data affected (new tables only, new columns with defaults) | Skip — Drizzle handles DDL |
| **Data-only** | Drizzle handles DDL, but a standalone data operation is also needed (cleanup, backfill nullable column, one-time DELETE/UPDATE) | Collect SQL → run against production in Phase 5.1 |
| **Simple** | DDL changes that Drizzle can't handle alone (column renames, type changes, backfill + NOT NULL constraint) | Write full migration script with Drizzle journal inserts (Phase 3) |
| **Complex** | Data transformation logic, ambiguous mappings, potential data loss | **STOP** and discuss with user |

**Key distinction:** "Data-only" means Drizzle's generated migrations are sufficient for the schema changes — the extra SQL is a data operation that runs *after* Drizzle migrations complete at deploy startup. "Simple" means the DDL itself needs manual handling (Drizzle's generated SQL would fail or produce wrong results on existing data).

### 2.4 Handle Data-only Migrations

For entries classified as Data-only:

1. Write the SQL statements to a file at `_migrations/data-YYYYMMDD.sql`
2. Use data-agnostic SQL (derive values from existing DB content, never hardcode user data)
3. These will be executed against production in Phase 5.1 **after** the deploy completes and Drizzle migrations run

### 2.5 Handle Complex Migrations

If any migration is classified as Complex:

1. Present the issue to the user:
   ```
   ## Migration Requires Discussion

   **Entry:** [MIGRATIONS.md entry]
   **Problem:** [why it's complex]
   **Options:**
   1. [Option A with trade-offs]
   2. [Option B with trade-offs]
   3. Skip this migration (manual intervention needed post-deploy)

   What would you like to do?
   ```

2. Wait for user input before proceeding.

## Phase 3: Write & Validate Migration

Only applies if Phase 2 found migrations to execute.

### 3.1 Identify Covered Drizzle Migrations

Check which Drizzle migration files (consult CLAUDE.md for migration directory location, typically `drizzle/`) are new since `release`:

```bash
git diff origin/release..HEAD --name-only -- <migration-dir>/
```

Read the SQL content of each new Drizzle migration file. The manual migration script must cover **all DDL and data changes** so these Drizzle files don't need to run again. After the manual script executes, we mark these migrations as applied in Drizzle's journal table.

### 3.2 Write Migration SQL

Create a migration file at `_migrations/release-YYYYMMDD.sql` with:
- Transaction wrapper (`BEGIN; ... COMMIT;`)
- **All DDL changes** from the covered Drizzle migration files (use safe idioms: `IF NOT EXISTS`, nullable-then-backfill-then-NOT-NULL)
- **All data transformations** from MIGRATIONS.md entries
- **Drizzle journal inserts** to mark covered migrations as applied
- Comments explaining each operation

**PRIVACY RULE: Never hardcode user data in migration SQL.** The migration file lives in gitignored `_migrations/`, but always write data-agnostic SQL that derives values from existing database content (e.g., `SELECT DISTINCT email FROM sessions`) rather than hardcoding emails, names, or other personal data. This makes the migration work for any data state and avoids accidental leaks.

**Drizzle journal compatibility.** Drizzle decides which migrations to run by checking the `created_at` of the **most recent** row in `"drizzle"."__drizzle_migrations"` and running all migrations with a `when` timestamp (from `drizzle/meta/_journal.json`) greater than that value. The manual script must insert journal rows so Drizzle skips the covered migrations at deploy startup. These internals are verified in Phase 1.4 against the exact `node_modules` code that will run in production.

**Journal table details:**
- Schema-qualified: `"drizzle"."__drizzle_migrations"` (NOT `public`)
- Columns: `id` serial PK, `hash` text NOT NULL, `created_at` numeric
- `hash` = SHA-256 hex of the SQL file contents: `shasum -a 256 <migration-dir>/XXXX.sql | cut -d' ' -f1`
- `created_at` = the `when` value from the corresponding entry in `drizzle/meta/_journal.json` (epoch milliseconds)

**Important:** Drizzle only compares `created_at` timestamps, not hashes, to decide what to run. But always insert correct hashes for integrity.

For a full example migration script, read [references/migration-example.md](references/migration-example.md) if it exists.

### 3.3 Validate Against Backup

Restore the production backup to a local test database and run the migration:

1. Restore backup:
   ```bash
   docker compose up -d
   docker compose exec -T db psql -U postgres -c "DROP DATABASE IF EXISTS migration_test;"
   docker compose exec -T db psql -U postgres -c "CREATE DATABASE migration_test;"
   # Adjust pg_restore path based on OS (use `which pg_restore` or check common paths)
   pg_restore \
     --dbname="postgresql://postgres:postgres@localhost:5432/migration_test" \
     --no-owner --no-privileges \
     _migrations/backup-*.dump
   ```

2. Run the migration:
   ```bash
   # Adjust psql path based on OS
   psql \
     "postgresql://postgres:postgres@localhost:5432/migration_test" \
     -f _migrations/release-YYYYMMDD.sql
   ```

3. Verify the migration succeeded (no errors)

4. Verify Drizzle journal entries were inserted:
   ```bash
   docker compose exec -T db psql -U postgres -d migration_test \
     -c "SELECT id, hash, created_at FROM drizzle.__drizzle_migrations ORDER BY id;"
   ```
   Confirm the new entries appear for the covered migrations with correct `created_at` values matching `drizzle/meta/_journal.json`.

5. Clean up:
   ```bash
   docker compose exec -T db psql -U postgres -c "DROP DATABASE migration_test;"
   ```

If local Docker is not available, show the migration SQL to the user and ask for manual approval.

### 3.4 Show Migration to User

Regardless of validation, display the full migration SQL to the user for review:

```
## Migration SQL (will be applied to production)

[SQL content]

Validated against production backup: [yes/no]
Drizzle journal entries: [list of covered migration files]
Proceed with release?
```

Wait for user confirmation.

## Phase 4: Backup Production Database

### 4.1 Create Backup Directory

```bash
mkdir -p _migrations
```

### 4.2 Get Production Database URL

Read CLAUDE.md to discover the deployment platform and environment variable names. For Railway:

```bash
railway run -e production -s <service-name> printenv DATABASE_PUBLIC_URL
```

Adjust command based on deployment platform discovered from CLAUDE.md.

If the command fails or returns empty, **STOP**: "Cannot connect to production database. Check deployment configuration and database URL variable."

### 4.3 Stop Production Service (if migration needed)

If migration SQL was written in Phase 3, stop the production app service **before** taking the backup. This ensures the backup captures every last write — no data can be lost between backup and migration.

Consult CLAUDE.md for the deployment platform and stop command. For Railway:

```bash
railway down -y -e production -s <service-name>
```

If the command fails, **STOP**: "Cannot stop production service. Check deployment CLI configuration."

**Note:** Stopping the service removes the most recent deployment — the app stops serving requests. The database is a separate service and continues running. No data is lost. Pushing to `release` later creates a new deployment that brings the service back.

If no migration is needed, skip this step — the service stays running during backup and merge.

### 4.4 Dump Production Database

```bash
# Adjust pg_dump path based on OS
pg_dump "$DATABASE_PUBLIC_URL" \
  --format=custom \
  --no-owner \
  --no-privileges \
  -f _migrations/backup-$(date +%Y%m%d-%H%M%S).dump
```

Verify the dump file was created and is non-empty:

```bash
ls -lh _migrations/backup-*.dump | tail -1
```

If dump fails, **STOP**: "Database backup failed. Do not proceed without a backup."

Report: "Production backup saved to `_migrations/backup-YYYYMMDD-HHMMSS.dump` (X KB)"

## Phase 5: Execute Release

### 5.1 Apply Migrations to Production

**Full migration script (Simple complexity):**

If a migration script was written in Phase 3, run it using the URL from Phase 4.2 **before** merging to release (service is already stopped):

```bash
# Adjust psql path based on OS
psql "$DATABASE_PUBLIC_URL" -f _migrations/release-YYYYMMDD.sql
```

If the migration fails, **STOP**: "Migration failed on production. Service is down. Investigate before proceeding. Backup available at `_migrations/backup-*.dump`. To restore service, push current `release` branch again or redeploy from deployment dashboard."

**Data-only SQL (Data-only complexity):**

If data-only SQL was written in Phase 2.4, run it **after** merging to release and confirming the deploy succeeded (Drizzle migrations must run first at deploy startup). Wait for the deployment to complete, then:

```bash
# Adjust psql path based on OS
psql -d "$DATABASE_PUBLIC_URL" -f _migrations/data-YYYYMMDD.sql
```

If no migrations or data operations are needed, skip this step.

### 5.2 Clear MIGRATIONS.md

Reset `MIGRATIONS.md` to its empty template:

```markdown
# Pending Production Migrations

Log potential production data migrations here during development. These notes are assessed and implemented by the `push-to-production` skill when promoting `main` to `release`.

**Format:** Describe what changed and what data is affected. Simple reference SQL is fine; do not write full migration scripts — those are built at release time.

<!-- Add entries below this line -->
```

### 5.3 Update Version and Changelog

**Determine version** (follows [Semantic Versioning 2.0.0](https://semver.org/)):

1. Read `CHANGELOG.md` and extract the current version from the first `## [x.y.z]` header
2. If `<arguments>` contains a version (e.g., `2.0.0`):
   - Validate it's valid semver (X.Y.Z)
   - Validate it's strictly higher than current version
   - If invalid, **STOP**: "Invalid version. Must be higher than current [current]."
3. If no argument, **deduce the bump from the commits being promoted** (from Phase 1.8):
   - **MAJOR** (`x+1.0.0`): Incompatible/breaking changes — removed or renamed API routes, changed API response shapes, DB schema changes that break existing clients, removed features
   - **MINOR** (`x.y+1.0`): Backward-compatible new functionality — new screens, new API endpoints, new features, significant UI additions
   - **PATCH** (`x.y.z+1`): Backward-compatible bug fixes — bug fixes, UI tweaks, refactoring, performance improvements, documentation, dependency updates
   - When commits span multiple categories, use the **highest** bump level (MAJOR > MINOR > PATCH)
   - Show the user which bump level was chosen and why, so they can override if they disagree

**Write changelog entry** (follows [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/)):

See [references/changelog-guidelines.md](references/changelog-guidelines.md) if it exists for full INCLUDE/EXCLUDE criteria and writing style rules.

**Process:**

1. Review the commit list from Phase 1.8
2. Filter out purely internal changes (they get zero entries)
3. Move any items from the `## [Unreleased]` section into the new version entry
4. Write a `## [version] - YYYY-MM-DD` entry, grouping changes under these section headers (omit empty sections):
   - `### Added` — new features, new screens
   - `### Changed` — changes to existing functionality, UI improvements
   - `### Deprecated` — features that will be removed in a future release
   - `### Removed` — removed features
   - `### Fixed` — bug fixes
   - `### Security` — security-related changes, vulnerability fixes
5. Group minor fixes into single items (e.g., "Minor bug fixes" or "Minor UI polish")
6. Keep each section concise — aim for 3-8 items total across all sections
7. Insert the new entry between `## [Unreleased]` and the previous version (keep Unreleased section empty)
8. Update the comparison links at the bottom of the file:
   - Discover the repository URL from git remote or CLAUDE.md
   - `[Unreleased]` link: compare new version tag to HEAD
   - New version link: compare previous version tag to new version tag
   - Format: `[Unreleased]: https://github.com/<owner>/<repo>/compare/vNEW...HEAD`
   - Format: `[NEW]: https://github.com/<owner>/<repo>/compare/vOLD...vNEW`

**Update package.json:**

Edit `package.json` to set `"version"` to the new version string.

### 5.4 Commit and Push to Main

Stage and commit all release housekeeping files:

```bash
git add MIGRATIONS.md CHANGELOG.md package.json
git commit -m "release: v<version>"
git push origin main
```

### 5.5 Merge Main to Release

```bash
git checkout release
git pull origin release
git merge origin/main --no-edit
git push origin release
```

If merge conflicts occur, **STOP** and tell the user to resolve them manually.

**Note:** Pushing to `release` triggers auto-deploy (if configured). If the service was stopped in Phase 4.3, the deploy will bring it back up automatically. Drizzle runs at startup, sees the covered migrations already in `__drizzle_migrations`, and skips them.

### 5.6 Tag Release

Create an annotated git tag on the `release` branch and push it:

```bash
git tag -a "v<version>" -m "v<version>"
git push origin "v<version>"
git checkout main
```

### 5.7 Create GitHub Release

Create a GitHub Release from the tag pushed in Phase 5.6. The release notes come from the changelog entry written in Phase 5.3.

**Extract release notes** from `CHANGELOG.md` — the content between the new `## [version]` header and the next `## [` header (excluding both headers). This is the same section written in Phase 5.3.

**Create the release:**

```bash
gh release create "v<version>" --title "v<version>" --notes "<release-notes>" --verify-tag
```

**Flags reference:**
- `--verify-tag` — Abort if the tag doesn't exist on the remote (safety check)
- `--title` — Release title (use the tag name, e.g., `v1.12.0`)
- `--notes` — Release notes body (the changelog section content with `### Added`, `### Fixed`, etc.)
- Do NOT use `--latest` — let GitHub auto-detect based on semver (default behavior is correct)
- Do NOT use `--draft` or `--prerelease` — all releases from this skill are production releases

**Error handling:** If `gh release create` fails, **do NOT stop the release**. Log a warning in the Phase 6 report:
```
**Warning:** GitHub Release creation failed: [error message]. Create manually with:
gh release create "v<version>" --title "v<version>" --notes-file <(extract from CHANGELOG.md) --verify-tag
```

The git tag and deploy already succeeded — the GitHub Release is cosmetic and can be created manually later.

## Phase 6: Post-Release

### 6.1 Move Done Issues to Released

Transition all Linear issues in "Done" to "Released" now that the code is live in production.

1. Query all issues in Done state:
   ```
   mcp__linear__list_issues with team: "<discovered-team-name>", state: "Done"
   ```

2. Look up the Released state UUID using `mcp__linear__list_issue_statuses` with team name. Find the status with `name: "Released"` (or similar — check what states exist in the team).

3. For each issue found, transition to Released using the **state UUID** (both Done and Released are `type: completed` — passing by name could silently no-op):
   ```
   mcp__linear__update_issue with id: <issue-id>, state: "<released-state-uuid>"
   ```

4. Collect the list of moved issues (identifier + title) for the report.

If no issues are in Done, that's fine — skip silently.

If the Linear MCP is unavailable (tools fail), **do not STOP** — log a warning in the report and continue. The release itself succeeded; issue state is cosmetic.

### 6.2 Report

```
## Release Complete

**Version:** X.Y.Z
**Promoted:** main → release
**Commits:** N commits
**Backup:** _migrations/backup-YYYYMMDD-HHMMSS.dump
**Migration:** [Applied successfully | No migration needed]
**Data operations:** [Applied successfully (list queries) | None]
**GitHub Release:** [Created | Failed (see warning above)]

### Issues Released
[List of PROJ-xxx: title moved from Done → Released, or "None"]

### Environment Variable Changes
[List any env var renames/additions from MIGRATIONS.md, or "None"]

### Next Steps
- Monitor deployment at production
- Verify production at [production URL from CLAUDE.md]
- Check deploy logs if issues arise
```

### 6.3 Remind About Env Vars

If MIGRATIONS.md mentioned any environment variable changes, remind the user:

```
**ACTION REQUIRED:** Update these environment variables in production before the deploy completes:
- [OLD_VAR → NEW_VAR]
- [NEW_VAR=value to add]
```

## Error Handling

| Situation | Action |
|-----------|--------|
| Not on `main` | STOP — switch to main first |
| Dirty working tree | STOP — commit or stash |
| Behind remote | STOP — pull latest |
| Build/tests fail | STOP — fix before releasing |
| No commits to promote | STOP — nothing to do |
| DB backup fails | STOP — never release without backup |
| Complex migration | STOP — discuss with user |
| Migration SQL fails locally | STOP — fix migration before proceeding |
| Migration fails on production | STOP — service is down, investigate, backup available, restore service |
| Data-only SQL fails on production | Log the error — deploy already succeeded, Drizzle migrations are fine. Report to user and suggest manual fix |
| Cannot stop production service | STOP — check deployment CLI |
| Drizzle internals changed | STOP — read migrator source, update skill before proceeding |
| Merge conflicts | STOP — user resolves manually |
| Deployment CLI not available | STOP — install/login deployment CLI |
| pg_dump not found | STOP — install PostgreSQL client tools |
| Docker not installed | STOP — install Docker |
| Docker not running | Start Docker, then continue |
| Invalid/lower version argument | STOP — must be valid semver higher than current |
| GitHub Release creation fails | Warn in report — release succeeded, create manually later |

## Rules

- **Stop service before backup when migrating** — Prevents data loss between backup and migration; backup captures every last write
- **Never skip migration assessment** — Even if MIGRATIONS.md is empty, check the diff
- **Show migration SQL to user** — Always get explicit confirmation before applying
- **Stop → backup → migrate → deploy** — Stopping the service stops writes, backup is consistent, migration applies cleanly, push to `release` restarts the service
- **Manual migration covers Drizzle DDL + data + journal** — One atomic script does everything; Drizzle skips already-applied migrations at startup
- **Clear MIGRATIONS.md after release** — Reset to empty template on main
- **No co-author attribution** — Commit messages must NOT include `Co-Authored-By` tags
- **Never force-push** — Use normal merge only
- **Backup files stay local** — `_migrations/` is gitignored
- **Never hardcode user data in SQL** — Derive from existing DB content (SELECT DISTINCT, JOINs), never hardcode emails, names, or personal data
- **Semantic Versioning 2.0.0** — Version bumps follow semver rules: MAJOR for breaking changes, MINOR for new features, PATCH for bug fixes. Every release gets a CHANGELOG.md entry and matching package.json version
- **Never defer SQL to the user** — All data operations from MIGRATIONS.md must be executed by this skill as part of the release. Never tell the user to run SQL manually post-deploy.
- **Stop on any failure** — Better to abort than corrupt production

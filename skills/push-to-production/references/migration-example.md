# Migration SQL Example

This example covers two Drizzle migrations (DDL + data backfill + journal inserts) in a single transaction.

```sql
-- Migration: main → release (YYYY-MM-DD)
-- Source: MIGRATIONS.md entries
-- Covers Drizzle migrations: 0005_clever_calypso, 0006_silky_roughhouse

BEGIN;

-- Step 1: DDL from Drizzle 0005 (safe for existing data)
CREATE TABLE IF NOT EXISTS "users" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
  "email" text NOT NULL,
  "name" text,
  "created_at" timestamp with time zone DEFAULT now() NOT NULL,
  "updated_at" timestamp with time zone DEFAULT now() NOT NULL,
  CONSTRAINT "users_email_unique" UNIQUE("email")
);
ALTER TABLE "custom_foods" ADD COLUMN IF NOT EXISTS "user_id" uuid;
-- (nullable first — NOT NULL added after backfill)

-- Step 2: Data migration — derive from existing DB data, never hardcode
INSERT INTO users (id, email)
SELECT gen_random_uuid(), LOWER(email)
FROM (SELECT DISTINCT email FROM sessions) s
ON CONFLICT DO NOTHING;

UPDATE custom_foods SET user_id = u.id
FROM users u WHERE LOWER(custom_foods.email) = u.email AND custom_foods.user_id IS NULL;
-- ... same pattern for other tables ...

-- Step 3: Finalize DDL (NOT NULL, constraints, drops from 0006)
ALTER TABLE "custom_foods" ALTER COLUMN "user_id" SET NOT NULL;
-- ... FK constraints, unique constraints, column drops ...

-- Step 4: Mark Drizzle migrations as applied
-- hash: shasum -a 256 drizzle/<file>.sql | cut -d' ' -f1
-- created_at: "when" field from drizzle/meta/_journal.json for each entry
INSERT INTO "drizzle"."__drizzle_migrations" (hash, created_at)
VALUES
  ('<sha256-of-0005-file>', 1770483401568),
  ('<sha256-of-0006-file>', 1770483422458);

COMMIT;
```

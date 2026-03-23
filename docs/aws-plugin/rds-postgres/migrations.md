# AWS RDS Postgres Migrations

## Creating Migrations

Migrations are created using `gas db:migration:create <migration_name>`.

## Running Migrations

Migrations are ran automatically:

- On `gas dev:start`, after the dev Postgres container starts.
- On Postgres migration file changes in dev mode, after the database is reset.
- During deployment, against the remote RDS Postgres database, if migrations are configured in the resource's `params.ts` file.

## Example Migration

`gas/api/migrations/20240101000000_init.sql`:
```sql
-- migrate:up
DROP TABLE IF EXISTS "Book";
DROP TABLE IF EXISTS "Author";

CREATE TABLE IF NOT EXISTS "Author" (
  "id" VARCHAR(21) PRIMARY KEY,
  "name" VARCHAR(100) NULL
);

CREATE TABLE IF NOT EXISTS "Book" (
  "id" VARCHAR(21) PRIMARY KEY,
  "authorId" VARCHAR(21) NOT NULL,
  "title" VARCHAR(200) NULL
);

-- migrate:down
DROP TABLE IF EXISTS "Book";
DROP TABLE IF EXISTS "Author";
```

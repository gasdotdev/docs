# D1 Migrations

## Defining Migrations

Migrations are configured in the D1 resource's `params.ts` file using the `migrate` property.

`gas/db/params.ts`:
```ts
import { defineCfD1 } from '@gasdotdev/plugin-cloudflare/definitions';

export const rootDb = (() => {
	return defineCfD1({
		name: 'root-db',
		migrate:
			process.env.GAS_STAGE === 'production'
				? {
						onChange: './migrations',
					}
				: undefined,
		seed:
			process.env.GAS_STAGE === 'preview'
				? {
						onChange: ['./migrations', './src/**/*.ts'],
					}
				: undefined,
	} as const);
})();
```

## Creating Migrations

Migrations are created using `npm run migration:create --workspace=<resource_name> -- <migration_name>`.

## Running Migrations

Migrations are ran automatically:

- On `gas dev:start`, against the local D1 database.
- On D1 migration file changes in dev mode.
- During deployment, against the remote D1 database, if migrations are configured in the resource's `params.ts` file.

## Example Migration

`gas/db/migrations/20240101000000_init.sql`:
```sql
-- migrate:up
PRAGMA foreign_keys=off;

DROP TABLE IF EXISTS "Book";
DROP TABLE IF EXISTS "Author";

CREATE TABLE IF NOT EXISTS "Author" (
  "id" TEXT PRIMARY KEY,
  "name" TEXT NULL
);

CREATE TABLE IF NOT EXISTS "Book" (
  "id" TEXT PRIMARY KEY,
  "authorId" TEXT NOT NULL,
  "title" TEXT NULL
);

-- migrate:down
DROP TABLE IF EXISTS "Book";
DROP TABLE IF EXISTS "Author";
```

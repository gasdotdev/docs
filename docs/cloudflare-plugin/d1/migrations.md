# D1 Migrations

## Creating Migrations

You can use the D1 resource's `package.json` `migration:create` script to create a migration file. The `migrate:create` script will run the [`@gasdotdev/db`](../../db-package/main.md) `gas-db create` command. An empty time-stamped migration file will be placed in the resource's `./migrations` directory.

> npm run migration:create --workspace=root-db -- create_customer_table

*Note: When a D1 resource is first added, an empty init migration file will be included in the installation and is ready to use (e.g. `./gas/db/migrations/20251110125013_init_migration.sql`).*

## Migration Files

A migration file contains two blocks: `-- migrate:up` and `-- migrate:down`. `-- migrate:up` is used create tables, columns, and indexes. `-- migrate:down` is used to undo those changes.

```sql
-- migrate:up
PRAGMA foreign_keys=off;

DROP TABLE IF EXISTS "Category";
DROP TABLE IF EXISTS "Customer";
DROP TABLE IF EXISTS "Product";
DROP TABLE IF EXISTS "Order";

CREATE TABLE IF NOT EXISTS "Category" (
  "id" VARCHAR(21) PRIMARY KEY,
  "name" VARCHAR(100) NULL,
  "description" VARCHAR(500) NULL
);

CREATE TABLE IF NOT EXISTS "Customer" (
  "id" VARCHAR(21) PRIMARY KEY,
  "firstName" VARCHAR(50) NULL,
  "lastName" VARCHAR(50) NULL,
  "address" VARCHAR(200) NULL,
  "city" VARCHAR(100) NULL,
  "state" VARCHAR(100) NULL,
  "postalCode" VARCHAR(20) NULL,
  "country" VARCHAR(100) NULL
);

CREATE TABLE IF NOT EXISTS "Product" (
  "id" VARCHAR(21) PRIMARY KEY,
  "name" VARCHAR(200) NULL,
  "categoryId" VARCHAR(21) NOT NULL,
  "price" DECIMAL NOT NULL
);

CREATE TABLE IF NOT EXISTS "Order" (
  "id" VARCHAR(21) PRIMARY KEY,
  "customerId" VARCHAR(21) NULL,
  "date" INTEGER NOT NULL,
  "cost" DECIMAL NOT NULL
);

-- migrate:down

```

## Applying Migrations in Development

Migrations are automatically applied during development.

## Applying Migrations in Staging

Migrations in staging are ran automatically during deployment when a D1 resource's `migrate` param is set in its `func.ts`.

During deployment, the `migrate` param becomes a separate sub-resource (`root-db:migrate`) in the dependency graph. That ensures migrations run after the database is created but before Workers that depend on it.

When `onChange` files have changed since the last deployment, the deployment runs `npx gas-db up` from the D1 resource's directory.

`npx gas-db up` uses `@gasdotdev/db` and `@gasdotdev/db-cloudflare-d1-driver`.

`@gasdotdev/db`:
1. Compares the `./migrations` directory against the remote D1 database's `_migrations` table.
2. Applies any pending migrations in order via the `@gasdotdev/db-cloudflare-d1-driver`.

If `onChange` files haven't changed, the migrate processor is skipped entirely.

##### Example

In this example, a `root-db` D1 resource exists.

Its migrations run on `production` stage deployments whenever migration files change. Other stage deployments, like `preview`, would skip migrations entirely because the seeding process handles migrations instead.

`./gas/db/func.ts`:

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

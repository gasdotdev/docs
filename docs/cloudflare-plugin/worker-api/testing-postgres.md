```ts
import {
  startTestWorker,
  type TestWorker,
} from '@gasdotdev/plugin-cloudflare/test/worker';
import { after, before, describe, it } from 'node:test';
import assert from 'node:assert';
import { createORPCClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { RouterClient } from '@orpc/server';
import {
  PostgreSqlContainer,
  type StartedPostgreSqlContainer,
} from '@testcontainers/postgresql';
import { Client } from 'pg';
import { migrateUp } from '@gasdotdev/gas/postgres';
import { rootApi } from '../../func.ts';
import { router } from '../index.ts';
import { rootDb } from 'root-db/func.ts';

let pgContainer: StartedPostgreSqlContainer;
let pgClient: Client;
let testWorker: TestWorker<Awaited<typeof rootApi>>;
let rpcClient: RouterClient<typeof router>;

before(async () => {
  const {
    name: rootDbName,
    image: rootDbImage,
    migrationsPath: rootDbMigrationsPath,
    password: rootDbPassword,
    user: rootDbUser,
  } = await rootDb;

  pgContainer = await new PostgreSqlContainer(rootDbImage)
    .withUsername(rootDbUser)
    .withPassword(rootDbPassword!)
    .withDatabase(rootDbName)
    .start();

  await migrateUp({
    connectionString: pgContainer.getConnectionUri(),
    migrationsPath: rootDbMigrationsPath,
  });

  pgClient = new Client({ connectionString: pgContainer.getConnectionUri() });
  await pgClient.connect();

  const { rows } = await pgClient.query(
    `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' ORDER BY table_name`,
  );
  console.log(
    'tables:',
    rows.map((r) => r.table_name),
  );

  testWorker = await startTestWorker({
    resourceParams: await rootApi,
    hyperdrive: { [rootDbName]: pgContainer.getConnectionUri() },
  });

  rpcClient = createORPCClient(new RPCLink({ url: `${testWorker.url}/rpc` }));
});

after(async () => {
  await pgClient.end();
  await testWorker.stop();
  await pgContainer.stop();
});

describe('root-api:hello', () => {
  it('should return hello world', async () => {
    const result = await rpcClient.hello();
    assert.deepStrictEqual(result, { message: 'hello world' });
  });
});
```

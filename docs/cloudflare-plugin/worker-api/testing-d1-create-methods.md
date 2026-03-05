# Worker API - Testing - D1 - Create Methods

## Example

`gas/api/src/test/authors.test.ts`:
```ts
import {
	startTestWorker,
	type TestWorker,
} from '@gasdotdev/plugin-cloudflare/test/worker';
import { after, before, it, describe } from 'node:test';
import { rootApi } from '../../func.ts';
import assert from 'node:assert';
import { faker } from '@faker-js/faker';
import { createORPCClient, safe } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { RouterClient } from '@orpc/server';
import { ORPCError } from '@orpc/server';
import { router } from '../index.ts';
import { createAuthor } from 'root-db/author.ts';

let testWorker: TestWorker<typeof rootApi>;
let client: RouterClient<typeof router>;

before(async () => {
	testWorker = await startTestWorker({
		resourceParams: rootApi,
	});

	const link = new RPCLink({
		url: `${testWorker.url}/rpc`,
	});

	client = createORPCClient(link);
});

after(async () => {
	await testWorker.stop();
});

describe('root-api:authors:create', () => {
	it('should create an author', async () => {
		const author = {
			name: faker.person.fullName(),
		};

		const result = await client.authors.create(author);

		assert.strictEqual(result.name, author.name);
		assert.ok(result.id);

		await deleteAuthor({
			db: testWorker.d1Databases['root-db'],
			id: result.id,
		});
	});

	it('should return 400 for invalid author data', async () => {
		const invalidAuthorData = {
			name: 123 as any,
		};

		const [error, data] = await safe(client.authors.create(invalidAuthorData));

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

# Worker API - Testing - D1 - Get Methods

## Example

`gas/api/src/test/books.test.ts`:
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
import { createBook } from 'root-db/book.ts';

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

describe('root-api:books:get', () => {
  it('should get an existing book', async () => {
		const bookData = {
			authorId: testAuthorId,
			title: faker.lorem.words({ min: 2, max: 5 }),
		};

		const createResult = await createBook({
			db: testWorker.d1Databases['root-db'],
			book: bookData,
		});

		assert(!createResult.err);

		const getResult = await client.books.get({ id: createResult.id });

		assert.strictEqual(getResult.id, createResult.id);
		assert.strictEqual(getResult.authorId, bookData.authorId);
		assert.strictEqual(getResult.title, bookData.title);

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: createResult.id,
		});
	});

	it('should return 404 for non-existent book', async () => {
		const [error, data] = await safe(
			client.books.get({ id: 'non-existent-id' }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 404);
		assert.ok(error.message);
	});

	it('should return 400 for invalid book ID', async () => {
		const [error, data] = await safe(
			client.books.get({ id: 123 as any }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

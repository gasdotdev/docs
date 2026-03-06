# Worker API - Testing - D1 - Create Methods

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

describe('root-api:books:create', () => {
	it('should create a valid book', async () => {
		const book = {
			authorId: testAuthorId,
			title: faker.lorem.words({ min: 2, max: 5 }),
		};

		const result = await client.books.create(book);

		assert.strictEqual(result.authorId, book.authorId);
		assert.strictEqual(result.title, book.title);
		assert.ok(result.id);

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: result.id,
		});
	});

	it('should return 400 for invalid author ID', async () => {
		const book = {
			authorId: 'non-existent-author',
			title: faker.lorem.words({ min: 2, max: 5 }),
		};

		const [error, data] = await safe(client.books.create(book));

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.ok(error.message);
	});

	it('should return 400 for invalid book data', async () => {
		const invalidBookData = {
			authorId: 123 as any,
			title: faker.lorem.words({ min: 2, max: 5 }),
		};

		const [error, data] = await safe(client.books.create(invalidBookData));

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

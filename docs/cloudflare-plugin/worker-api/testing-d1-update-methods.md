# Worker API - Testing - D1 - Update Methods

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
import { createBook, deleteBook } from 'root-db/book.ts';

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

describe('root-api:books:update', () => {
  it('should update existing book', async () => {
		const createResult = await createBook({
			db: testWorker.d1Databases['root-db'],
			book: {
				authorId: testAuthorId,
				title: 'Original Title',
			},
		});

		assert(!createResult.err);

		const updateResult = await client.books.update({
			id: createResult.id,
			authorId: testAuthorId,
			title: 'Updated Title',
		});

		assert.strictEqual(updateResult.id, createResult.id);
		assert.strictEqual(updateResult.title, 'Updated Title');

		const getResult = await client.books.get({ id: createResult.id });
		assert.strictEqual(getResult.title, 'Updated Title');

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: createResult.id,
		});
	});

	it('should return 404 when updating non-existent book', async () => {
		const [error, data] = await safe(
			client.books.update({
				id: 'non-existent-id',
				authorId: testAuthorId,
				title: 'Updated Title',
			}),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 404);
		assert.ok(error.message);
	});

	it('should return 400 for invalid author ID on update', async () => {
		const createResult = await createBook({
			db: testWorker.d1Databases['root-db'],
			book: {
				authorId: testAuthorId,
				title: faker.lorem.words({ min: 2, max: 5 }),
			},
		});

		assert(!createResult.err);

		const [error, data] = await safe(
			client.books.update({
				id: createResult.id,
				authorId: 'non-existent-author',
				title: 'Updated Title',
			}),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.ok(error.message);

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: createResult.id,
		});
	});

	it('should return 400 for invalid book data', async () => {
		const createResult = await createBook({
			db: testWorker.d1Databases['root-db'],
			book: {
				authorId: testAuthorId,
				title: faker.lorem.words({ min: 2, max: 5 }),
			},
		});

		assert(!createResult.err);

		const [error, data] = await safe(
			client.books.update({
				id: createResult.id,
				authorId: 123 as any,
				title: 'Updated Title',
			}),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: createResult.id,
		});
	});
});
```

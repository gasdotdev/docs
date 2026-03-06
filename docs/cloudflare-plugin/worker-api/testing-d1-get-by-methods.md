# Worker API - Testing - D1 - Get By Methods

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

describe('root-api:books:getBy', () => {
  it('should get books by author ID', async () => {
		const createResult = await createBook({
			db: testWorker.d1Databases['root-db'],
			book: {
				authorId: testAuthorId,
				title: faker.lorem.words({ min: 2, max: 5 }),
			},
		});

		assert(!createResult.err);

		const result = await client.books.getByAuthor({
			authorId: testAuthorId,
			page: 1,
			limit: 10,
		});

		assert.ok(result.books.length >= 1);
		assert.ok(
			result.books.every((book) => book.authorId === testAuthorId),
		);

		await deleteBook({
			db: testWorker.d1Databases['root-db'],
			id: createResult.id,
		});
	});

	it('should return empty array for author with no books', async () => {
		const authorResult = await createAuthor({
			db: testWorker.d1Databases['root-db'],
			author: { name: 'Empty Author' },
		});

		assert(!authorResult.err);

		const result = await client.books.getByAuthor({
			authorId: authorResult.id,
			page: 1,
			limit: 10,
		});

		assert.strictEqual(result.books.length, 0);
		assert.strictEqual(result.pagination.total, 0);

		await deleteAuthor({
			db: testWorker.d1Databases['root-db'],
			id: authorResult.id,
		});
	});

	it('should handle pagination for author books', async () => {
		const bookIds = [];

		for (let i = 0; i < 8; i++) {
			const createResult = await createBook({
				db: testWorker.d1Databases['root-db'],
				book: {
					authorId: testAuthorId,
					title: `Paginated Book ${i + 1}`,
				},
			});

			if (!createResult.err) {
				bookIds.push(createResult.id);
			}
		}

		const page1Result = await client.books.getByAuthor({
			authorId: testAuthorId,
			page: 1,
			limit: 5,
		});

		assert.strictEqual(page1Result.books.length, 5);
		assert.ok(page1Result.pagination.total >= 8);
		assert.strictEqual(page1Result.pagination.hasNext, true);

		const page2Result = await client.books.getByAuthor({
			authorId: testAuthorId,
			page: 2,
			limit: 5,
		});

		assert.ok(page2Result.books.length >= 3);
		assert.strictEqual(page2Result.pagination.hasNext, false);

		for (const bookId of bookIds) {
			await deleteBook({
				db: testWorker.d1Databases['root-db'],
				id: bookId,
			});
		}
	});
});
```

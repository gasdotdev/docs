# Worker API - Testing - D1 - Get All Methods

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

describe('root-api:books:getAll', () => {
  async function createTestBooks(count: number) {
		const books = [];
		for (let i = 0; i < count; i++) {
			const createResult = await createBook({
				db: testWorker.d1Databases['root-db'],
				book: {
					authorId: testAuthorId,
					title: `Book ${i + 1}`,
				},
			});
			if (!createResult.err) {
				books.push(createResult);
			}
		}
		return books;
	}

	async function cleanupTestBooks(bookIds: string[]) {
		for (const id of bookIds) {
			await deleteBook({
				db: testWorker.d1Databases['root-db'],
				id,
			});
		}
	}

	it('should handle an empty table gracefully', async () => {
		const result = await client.books.getAll({ page: 100, limit: 10 });

		assert.strictEqual(result.books.length, 0);
		assert.strictEqual(result.pagination.page, 100);
		assert.strictEqual(result.pagination.hasNext, false);
	});

	it('should get all books with default pagination', async () => {
		const testBooks = await createTestBooks(15);
		const testBookIds = testBooks.map((book) => book.id);

		const result = await client.books.getAll({});

		assert.strictEqual(result.pagination.page, 1);
		assert.strictEqual(result.pagination.limit, 10);
		assert.strictEqual(result.books.length, 10);
		assert.ok(result.pagination.total >= 15);
		assert.strictEqual(result.pagination.hasPrev, false);

		await cleanupTestBooks(testBookIds);
	});

	it('should paginate results correctly', async () => {
		const testBooks = await createTestBooks(15);
		const testBookIds = testBooks.map((book) => book.id);

		// Get first page
		const page1Result = await client.books.getAll({ page: 1, limit: 5 });

		assert.strictEqual(page1Result.books.length, 5);
		assert.strictEqual(page1Result.pagination.page, 1);
		assert.strictEqual(page1Result.pagination.limit, 5);
		assert.strictEqual(page1Result.pagination.hasNext, true);
		assert.strictEqual(page1Result.pagination.hasPrev, false);

		// Get second page
		const page2Result = await client.books.getAll({ page: 2, limit: 5 });

		assert.strictEqual(page2Result.books.length, 5);
		assert.strictEqual(page2Result.pagination.page, 2);
		assert.strictEqual(page2Result.pagination.limit, 5);
		assert.strictEqual(page2Result.pagination.hasPrev, true);

		// Verify no overlap between pages
		const page1Ids = page1Result.books.map((b) => b.id);
		const page2Ids = page2Result.books.map((b) => b.id);
		const allIds = [...page1Ids, ...page2Ids];
		assert.strictEqual(new Set(allIds).size, 10); // All unique

		await cleanupTestBooks(testBookIds);
	});

	it('should handle pagination beyond available data', async () => {
		const testBooks = await createTestBooks(5);
		const testBookIds = testBooks.map((book) => book.id);

		const result = await client.books.getAll({ page: 100, limit: 10 });

		assert.strictEqual(result.books.length, 0);
		assert.strictEqual(result.pagination.page, 100);
		assert.strictEqual(result.pagination.limit, 10);
		assert.strictEqual(result.pagination.hasNext, false);
		assert.strictEqual(result.pagination.hasPrev, true);

		await cleanupTestBooks(testBookIds);
	});

	it('should use defaults when no params are provided', async () => {
		const testBooks = await createTestBooks(3);
		const testBookIds = testBooks.map((book) => book.id);

		const result = await client.books.getAll({});

		assert.strictEqual(result.pagination.page, 1);
		assert.strictEqual(result.pagination.limit, 10);

		await cleanupTestBooks(testBookIds);
	});

	it('should reject page < 1', async () => {
		const [error, data] = await safe(
			client.books.getAll({ page: 0, limit: 10 }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});

	it('should reject limit < 1', async () => {
		const [error, data] = await safe(
			client.books.getAll({ page: 1, limit: 0 }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});

	it('should reject limit > 100', async () => {
		const [error, data] = await safe(
			client.books.getAll({ page: 1, limit: 101 }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

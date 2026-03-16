# Worker API - Testing - Postgres

The preferred pattern for testing a Worker API that uses Postgres.

## Setup

`gas/api/src/books.test.ts`:
```ts
import {
	startTestWorker,
	type TestWorker,
} from '@gasdotdev/plugin-cloudflare/test/worker';
import { after, before, it, describe } from 'node:test';
import { rootApi } from '../../params.ts';
import assert from 'node:assert';
import { faker } from '@faker-js/faker';
import { createORPCClient, safe } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { RouterClient } from '@orpc/server';
import { ORPCError } from '@orpc/server';
import {
	PostgreSqlContainer,
	type StartedPostgreSqlContainer,
} from '@testcontainers/postgresql';
import { Client } from 'pg';
import { migrateUp } from '@gasdotdev/gas/postgres';
import { router } from '../index.ts';
import { getRootDbParams } from 'root-db/params.ts';
import {
	createBook,
	deleteBook,
} from 'root-db/book.ts';
import {
	createAuthor,
	deleteAuthor,
} from 'root-db/author.ts';

let pgContainer: StartedPostgreSqlContainer;
let pgClient: Client;
let testWorker: TestWorker<typeof rootApi>;
let client: RouterClient<typeof router>;

before(async () => {
	const rootDbParams = await getRootDbParams();

	pgContainer = await new PostgreSqlContainer(rootDbParams.image)
		.withUsername(rootDbParams.user)
		.withPassword(rootDbParams.password)
		.withDatabase(rootDbParams.name)
		.withExposedPorts({
			container: rootDbParams.containerPort,
			host: rootDbParams.port,
		})
		.start();

	await migrateUp({
		connectionString: pgContainer.getConnectionUri(),
		migrationsPath: rootDbParams.migrationsPath,
	});

	pgClient = new Client({
		connectionString: pgContainer.getConnectionUri(),
	});
	await pgClient.connect();

	testWorker = await startTestWorker({
		resourceParams: rootApi,
	});

	const link = new RPCLink({
		url: `${testWorker.url}/rpc`,
	});

	client = createORPCClient(link);
});

after(async () => {
	await pgClient.end();
	await testWorker.stop();
	await pgContainer.stop();
});
```

## Example Create Methods

`gas/api/src/books.test.ts`:
```ts
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
			db: pgClient,
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

## Example Get Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:get', () => {
  it('should get an existing book', async () => {
		const bookData = {
			authorId: testAuthorId,
			title: faker.lorem.words({ min: 2, max: 5 }),
		};

		const createResult = await createBook({
			db: pgClient,
			book: bookData,
		});

		assert(!createResult.err);

		const getResult = await client.books.get({ id: createResult.id });

		assert.strictEqual(getResult.id, createResult.id);
		assert.strictEqual(getResult.authorId, bookData.authorId);
		assert.strictEqual(getResult.title, bookData.title);

		await deleteBook({
			db: pgClient,
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

## Example Get All Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:getAll', () => {
  async function createTestBooks(count: number) {
		const books = [];
		for (let i = 0; i < count; i++) {
			const createResult = await createBook({
				db: pgClient,
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
				db: pgClient,
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

## Example Get By Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:getBy', () => {
  it('should get books by author ID', async () => {
		const createResult = await createBook({
			db: pgClient,
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
			db: pgClient,
			id: createResult.id,
		});
	});

	it('should return empty array for author with no books', async () => {
		const authorResult = await createAuthor({
			db: pgClient,
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
			db: pgClient,
			id: authorResult.id,
		});
	});

	it('should handle pagination for author books', async () => {
		const bookIds = [];

		for (let i = 0; i < 8; i++) {
			const createResult = await createBook({
				db: pgClient,
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
				db: pgClient,
				id: bookId,
			});
		}
	});
});
```

## Example Search Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:search', () => {
  it('should find books by title', async () => {
		const createResult = await createBook({
			db: pgClient,
			book: {
				authorId: testAuthorId,
				title: 'Unique Searchable Title',
			},
		});

		assert(!createResult.err);

		const result = await client.books.search({
			query: 'Unique Searchable',
			page: 1,
			limit: 10,
		});

		assert.ok(result.books.length >= 1);
		assert.ok(
			result.books.some((book) =>
				book.title?.includes('Unique Searchable'),
			),
		);

		await deleteBook({
			db: pgClient,
			id: createResult.id,
		});
	});

	it('should return empty array for no matches', async () => {
		const result = await client.books.search({
			query: 'xyznonexistentquery',
			page: 1,
			limit: 10,
		});

		assert.strictEqual(result.books.length, 0);
		assert.strictEqual(result.pagination.total, 0);
	});

	it('should handle pagination for search results', async () => {
		const bookIds = [];

		for (let i = 0; i < 8; i++) {
			const createResult = await createBook({
				db: pgClient,
				book: {
					authorId: testAuthorId,
					title: `Searchable Pattern ${i + 1}`,
				},
			});

			if (!createResult.err) {
				bookIds.push(createResult.id);
			}
		}

		const page1Result = await client.books.search({
			query: 'Searchable Pattern',
			page: 1,
			limit: 5,
		});

		assert.strictEqual(page1Result.books.length, 5);
		assert.strictEqual(page1Result.pagination.hasNext, true);

		const page2Result = await client.books.search({
			query: 'Searchable Pattern',
			page: 2,
			limit: 5,
		});

		assert.strictEqual(page2Result.books.length, 3);
		assert.strictEqual(page2Result.pagination.hasNext, false);

		for (const bookId of bookIds) {
			await deleteBook({
				db: pgClient,
				id: bookId,
			});
		}
	});

	it('should return 400 for empty search query', async () => {
		const [error, data] = await safe(
			client.books.search({ query: '', page: 1, limit: 10 }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

## Example Update Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:update', () => {
  it('should update existing book', async () => {
		const createResult = await createBook({
			db: pgClient,
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
			db: pgClient,
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
			db: pgClient,
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
			db: pgClient,
			id: createResult.id,
		});
	});

	it('should return 400 for invalid book data', async () => {
		const createResult = await createBook({
			db: pgClient,
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
			db: pgClient,
			id: createResult.id,
		});
	});
});
```

## Example Delete Methods

`gas/api/src/books.test.ts`:
```ts
describe('root-api:books:delete', () => {
  it('should delete existing book', async () => {
		const createResult = await createBook({
			db: pgClient,
			book: {
				authorId: testAuthorId,
				title: faker.lorem.words({ min: 2, max: 5 }),
			},
		});

		assert(!createResult.err);

		const deleteResult = await client.books.delete({
			id: createResult.id,
		});

		assert.strictEqual(deleteResult.id, createResult.id);
		assert.strictEqual(deleteResult.changes, 1);

		const [error, data] = await safe(
			client.books.get({ id: createResult.id }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 404);
	});

	it('should return 404 for non-existent book', async () => {
		const [error, data] = await safe(
			client.books.delete({ id: 'non-existent-id' }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 404);
		assert.ok(error.message);
	});

	it('should return 400 for invalid book ID', async () => {
		const [error, data] = await safe(
			client.books.delete({ id: 123 as any }),
		);

		assert.strictEqual(data, undefined);
		assert.ok(error instanceof ORPCError);
		assert.strictEqual(error.status, 400);
		assert.ok(error.message);
	});
});
```

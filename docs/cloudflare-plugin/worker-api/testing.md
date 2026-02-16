# Worker API Testing

Tests can be ran for individual resources using this command: `npm run test --workspace=<resource_name>`

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
import { createAuthor, deleteAuthor } from 'root-db/author.ts';

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

describe('root-api:authors', () => {
	describe('create', () => {
		it('should create a valid author', async () => {
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

		it('should create author with null name', async () => {
			const author = {
				name: null,
			};

			const result = await client.authors.create(author);

			assert.strictEqual(result.name, null);
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

	describe('get', () => {
		it('should get existing author', async () => {
			const authorData = {
				name: faker.person.fullName(),
			};

			const createResult = await createAuthor({
				db: testWorker.d1Databases['root-db'],
				author: authorData,
			});

			assert(!createResult.err);

			const getResult = await client.authors.get({ id: createResult.id });

			assert.strictEqual(getResult.id, createResult.id);
			assert.strictEqual(getResult.name, authorData.name);

			await deleteAuthor({
				db: testWorker.d1Databases['root-db'],
				id: createResult.id,
			});
		});

		it('should return 404 for non-existent author', async () => {
			const nonExistentId = 'non-existent-id';

			const [error, data] = await safe(
				client.authors.get({ id: nonExistentId }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 404);
			assert.ok(error.message);
		});

		it('should return 400 for invalid author ID', async () => {
			const invalidId = 123 as any;

			const [error, data] = await safe(client.authors.get({ id: invalidId }));

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);
		});
	});

	describe('getAll', () => {
		async function createTestAuthors(count: number) {
			const authors = [];
			for (let i = 0; i < count; i++) {
				const createResult = await createAuthor({
					db: testWorker.d1Databases['root-db'],
					author: { name: `Author ${i + 1}` },
				});
				if (!createResult.err) {
					authors.push(createResult);
				}
			}
			return authors;
		}

		async function cleanupTestAuthors(authorIds: string[]) {
			for (const id of authorIds) {
				await deleteAuthor({
					db: testWorker.d1Databases['root-db'],
					id,
				});
			}
		}

		it('should handle empty table gracefully', async () => {
			const result = await client.authors.getAll({ page: 100, limit: 10 });

			assert.strictEqual(result.authors.length, 0);
			assert.strictEqual(result.pagination.page, 100);
			assert.strictEqual(result.pagination.hasNext, false);
		});

		it('should get all authors with default pagination', async () => {
			const testAuthors = await createTestAuthors(15);
			const testAuthorIds = testAuthors.map((author) => author.id);

			const result = await client.authors.getAll({});

			assert.strictEqual(result.pagination.page, 1);
			assert.strictEqual(result.pagination.limit, 10);
			assert.strictEqual(result.authors.length, 10);
			assert.ok(result.pagination.total >= 15);
			assert.strictEqual(result.pagination.hasPrev, false);

			await cleanupTestAuthors(testAuthorIds);
		});

		it('should paginate results correctly', async () => {
			const testAuthors = await createTestAuthors(15);
			const testAuthorIds = testAuthors.map((author) => author.id);

			// Get first page
			const page1Result = await client.authors.getAll({ page: 1, limit: 5 });

			assert.strictEqual(page1Result.authors.length, 5);
			assert.strictEqual(page1Result.pagination.page, 1);
			assert.strictEqual(page1Result.pagination.limit, 5);
			assert.strictEqual(page1Result.pagination.hasNext, true);
			assert.strictEqual(page1Result.pagination.hasPrev, false);

			// Get second page
			const page2Result = await client.authors.getAll({ page: 2, limit: 5 });

			assert.strictEqual(page2Result.authors.length, 5);
			assert.strictEqual(page2Result.pagination.page, 2);
			assert.strictEqual(page2Result.pagination.limit, 5);
			assert.strictEqual(page2Result.pagination.hasPrev, true);

			// Verify no overlap between pages
			const page1Ids = page1Result.authors.map((a) => a.id);
			const page2Ids = page2Result.authors.map((a) => a.id);
			const allIds = [...page1Ids, ...page2Ids];
			assert.strictEqual(new Set(allIds).size, 10); // All unique

			await cleanupTestAuthors(testAuthorIds);
		});

		it('should handle pagination beyond available data', async () => {
			const testAuthors = await createTestAuthors(5);
			const testAuthorIds = testAuthors.map((author) => author.id);

			const result = await client.authors.getAll({ page: 100, limit: 10 });

			assert.strictEqual(result.authors.length, 0);
			assert.strictEqual(result.pagination.page, 100);
			assert.strictEqual(result.pagination.limit, 10);
			assert.strictEqual(result.pagination.hasNext, false);
			assert.strictEqual(result.pagination.hasPrev, true);

			await cleanupTestAuthors(testAuthorIds);
		});

		it('should use defaults when no params provided', async () => {
			const testAuthors = await createTestAuthors(3);
			const testAuthorIds = testAuthors.map((author) => author.id);

			const result = await client.authors.getAll({});

			// Should use default values
			assert.strictEqual(result.pagination.page, 1);
			assert.strictEqual(result.pagination.limit, 10);

			await cleanupTestAuthors(testAuthorIds);
		});

		it('should reject page < 1', async () => {
			const [error, data] = await safe(
				client.authors.getAll({ page: 0, limit: 10 }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);
		});

		it('should reject limit < 1', async () => {
			const [error, data] = await safe(
				client.authors.getAll({ page: 1, limit: 0 }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);
		});

		it('should reject limit > 100', async () => {
			const [error, data] = await safe(
				client.authors.getAll({ page: 1, limit: 101 }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);
		});
	});

	describe('update', () => {
		it('should update existing author', async () => {
			const createResult = await createAuthor({
				db: testWorker.d1Databases['root-db'],
				author: { name: 'Original Name' },
			});

			assert(!createResult.err);

			const updateResult = await client.authors.update({
				id: createResult.id,
				name: 'Updated Name',
			});

			assert.strictEqual(updateResult.id, createResult.id);
			assert.strictEqual(updateResult.name, 'Updated Name');

			const getResult = await client.authors.get({ id: createResult.id });
			assert.strictEqual(getResult.name, 'Updated Name');

			await deleteAuthor({
				db: testWorker.d1Databases['root-db'],
				id: createResult.id,
			});
		});

		it('should return 404 when updating non-existent author', async () => {
			const [error, data] = await safe(
				client.authors.update({
					id: 'non-existent-id',
					name: 'Updated Name',
				}),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 404);
			assert.ok(error.message);
		});

		it('should return 400 for invalid author data', async () => {
			const createResult = await createAuthor({
				db: testWorker.d1Databases['root-db'],
				author: { name: faker.person.fullName() },
			});

			assert(!createResult.err);

			const [error, data] = await safe(
				client.authors.update({
					id: createResult.id,
					name: 123 as any,
				}),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);

			await deleteAuthor({
				db: testWorker.d1Databases['root-db'],
				id: createResult.id,
			});
		});
	});

	describe('delete', () => {
		it('should delete existing author', async () => {
			const createResult = await createAuthor({
				db: testWorker.d1Databases['root-db'],
				author: { name: faker.person.fullName() },
			});

			assert(!createResult.err);

			const deleteResult = await client.authors.delete({
				id: createResult.id,
			});

			assert.strictEqual(deleteResult.id, createResult.id);
			assert.strictEqual(deleteResult.changes, 1);

			const [error, data] = await safe(
				client.authors.get({ id: createResult.id }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 404);
		});

		it('should return 404 for non-existent author', async () => {
			const [error, data] = await safe(
				client.authors.delete({ id: 'non-existent-id' }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 404);
			assert.ok(error.message);
		});

		it('should return 400 for invalid author ID', async () => {
			const invalidId = 123 as any;

			const [error, data] = await safe(
				client.authors.delete({ id: invalidId }),
			);

			assert.strictEqual(data, undefined);
			assert.ok(error instanceof ORPCError);
			assert.strictEqual(error.status, 400);
			assert.ok(error.message);
		});
	});
});
```

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
import { createAuthor, deleteAuthor } from 'root-db/author.ts';
import { createBook, deleteBook } from 'root-db/book.ts';

let testWorker: TestWorker<typeof rootApi>;
let client: RouterClient<typeof router>;

// Shared author for FK references
let testAuthorId: string;

before(async () => {
	testWorker = await startTestWorker({
		resourceParams: rootApi,
	});

	const link = new RPCLink({
		url: `${testWorker.url}/rpc`,
	});

	client = createORPCClient(link);

	// Create a shared author for book FK references
	const authorResult = await createAuthor({
		db: testWorker.d1Databases['root-db'],
		author: { name: 'Test Author' },
	});

	assert(!authorResult.err);
	testAuthorId = authorResult.id;
});

after(async () => {
	await deleteAuthor({
		db: testWorker.d1Databases['root-db'],
		id: testAuthorId,
	});
	await testWorker.stop();
});

describe('root-api:books', () => {
	describe('create', () => {
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

		it('should create book with null title', async () => {
			const book = {
				authorId: testAuthorId,
				title: null,
			};

			const result = await client.books.create(book);

			assert.strictEqual(result.title, null);
			assert.strictEqual(result.authorId, testAuthorId);
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

	describe('get', () => {
		it('should get existing book', async () => {
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

	describe('getAll', () => {
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

		it('should handle empty table gracefully', async () => {
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

		it('should use defaults when no params provided', async () => {
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

	describe('getByAuthor', () => {
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

	describe('search', () => {
		it('should find books by title', async () => {
			const createResult = await createBook({
				db: testWorker.d1Databases['root-db'],
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
				db: testWorker.d1Databases['root-db'],
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
					db: testWorker.d1Databases['root-db'],
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
					db: testWorker.d1Databases['root-db'],
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

	describe('update', () => {
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

	describe('delete', () => {
		it('should delete existing book', async () => {
			const createResult = await createBook({
				db: testWorker.d1Databases['root-db'],
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
});
```

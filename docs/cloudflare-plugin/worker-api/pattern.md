# Worker API Pattern

- oRPC docs: https://orpc.dev/docs/getting-started
- oRPC OpenAPI docs: https://orpc.dev/docs/openapi/getting-started
- oRPC llms.txt: https://orpc.dev/llms.txt

`gas/api/src/index.ts`:
```ts
import { os } from '@orpc/server';
import { RPCHandler } from '@orpc/server/fetch';
import {
	createAuthor,
	createAuthorInput,
	getAuthor,
	getAuthorInput,
	getAllAuthors,
	getAllAuthorsInput,
	getAllAuthorsOutput,
	updateAuthor,
	updateAuthorInput,
	deleteAuthor,
	deleteAuthorInput,
	deleteAuthorOutput,
	AuthorSchema,
} from '../../db/src/author.ts';
import {
	createBook,
	createBookInput,
	getBook,
	getBookInput,
	getAllBooks,
	getAllBooksInput,
	getAllBooksOutput,
	getBooksByAuthor,
	getBooksByAuthorInput,
	searchBooks,
	searchBooksInput,
	updateBook,
	updateBookInput,
	deleteBook,
	deleteBookInput,
	deleteBookOutput,
	BookSchema,
} from '../../db/src/book.ts';
import { res, ERRS } from 'core';

const base = os.$context<{ env: Cloudflare.Env }>().errors(ERRS);

const rateLimit = base.middleware(async ({ next, errors }) => {
	return next();
});

const api = base.use(rateLimit);

// ==============================
// AUTHOR ROUTES
// ==============================

const createAuthorRoute = api
	.route({ method: 'POST' })
	.input(createAuthorInput)
	.output(AuthorSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createAuthor({
				db: context.env['root-db'],
				author: { ...input },
			}),
			errors,
		);
	});

const getAuthorRoute = api
	.route({ method: 'GET' })
	.input(getAuthorInput)
	.output(AuthorSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAuthor({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllAuthorsRoute = api
	.route({ method: 'GET' })
	.input(getAllAuthorsInput)
	.output(getAllAuthorsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllAuthors({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateAuthorRoute = api
	.route({ method: 'PUT' })
	.input(updateAuthorInput)
	.output(AuthorSchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...author } = input;
		return res(
			await updateAuthor({
				db: context.env['root-db'],
				id,
				author,
			}),
			errors,
		);
	});

const deleteAuthorRoute = api
	.route({ method: 'DELETE' })
	.input(deleteAuthorInput)
	.output(deleteAuthorOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteAuthor({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

// ==============================
// BOOK ROUTES
// ==============================

const createBookRoute = api
	.route({ method: 'POST' })
	.input(createBookInput)
	.output(BookSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createBook({
				db: context.env['root-db'],
				book: { ...input },
			}),
			errors,
		);
	});

const getBookRoute = api
	.route({ method: 'GET' })
	.input(getBookInput)
	.output(BookSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getBook({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllBooksRoute = api
	.route({ method: 'GET' })
	.input(getAllBooksInput)
	.output(getAllBooksOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllBooks({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getBooksByAuthorRoute = api
	.route({ method: 'GET' })
	.input(getBooksByAuthorInput)
	.output(getAllBooksOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getBooksByAuthor({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const searchBooksRoute = api
	.route({ method: 'GET' })
	.input(searchBooksInput)
	.output(getAllBooksOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await searchBooks({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateBookRoute = api
	.route({ method: 'PUT' })
	.input(updateBookInput)
	.output(BookSchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...book } = input;
		return res(
			await updateBook({
				db: context.env['root-db'],
				id,
				book,
			}),
			errors,
		);
	});

const deleteBookRoute = api
	.route({ method: 'DELETE' })
	.input(deleteBookInput)
	.output(deleteBookOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteBook({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

// ==============================
// ROUTER CONFIG
// ==============================

export const router = {
	authors: {
		create: createAuthorRoute,
		get: getAuthorRoute,
		getAll: getAllAuthorsRoute,
		update: updateAuthorRoute,
		delete: deleteAuthorRoute,
	},
	books: {
		create: createBookRoute,
		get: getBookRoute,
		getAll: getAllBooksRoute,
		getByAuthor: getBooksByAuthorRoute,
		search: searchBooksRoute,
		update: updateBookRoute,
		delete: deleteBookRoute,
	},
};

const rpcHandler = new RPCHandler(os.router(router), {});

export default {
	async fetch(request, env) {
		const rpcResult = await rpcHandler.handle(request, {
			context: {
				env,
			},
		});

		if (rpcResult.matched) {
			return rpcResult.response;
		}

		return new Response('Not Found', { status: 404 });
	},
} satisfies ExportedHandler<Cloudflare.Env>;

export type ApiRouter = typeof router;
```

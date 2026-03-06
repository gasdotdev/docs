# Worker API - Pattern - Postgres Connection

## Example

`gas/api/src/index.ts`:
```ts
import { os } from '@orpc/server';
import { RPCHandler } from '@orpc/server/fetch';
import { Client } from 'pg';
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

// ==============================
// MIDDLEWARE
// ==============================

const withDb = base.middleware(async ({ context, next }) => {
  const client = new Client({
      connectionString: context.env.HYPERDRIVE.connectionString,
  });
  await client.connect();
  try {
      return await next({
          context: { db: client },
      });
  } finally {
      await client.end();
  }
});

const rateLimit = base.middleware(async ({ next }) => {
  return next();
});

const api = base.use(rateLimit).use(withDb);

// ==============================
// BOOK ROUTES
// ==============================

const createBookRoute = api
  .route({ method: 'POST' })
  .input(createBookInput)
  .output(BookSchema)
  .handler(async ({ context, input, errors }) => {
      return res(
          await createBook({ db: context.db, book: { ...input } }),
          errors,
      );
  });

const getBookRoute = api
  .route({ method: 'GET' })
  .input(getBookInput)
  .output(BookSchema)
  .handler(async ({ context, input, errors }) => {
      return res(
          await getBook({ db: context.db, ...input }),
          errors,
      );
  });

const getAllBooksRoute = api
  .route({ method: 'GET' })
  .input(getAllBooksInput)
  .output(getAllBooksOutput)
  .handler(async ({ context, input, errors }) => {
      return res(
          await getAllBooks({ db: context.db }),
          errors,
      );
  });

const getBooksByAuthorRoute = api
  .route({ method: 'GET' })
  .input(getBooksByAuthorInput)
  .output(getAllBooksOutput)
  .handler(async ({ context, input, errors }) => {
      return res(
          await getBooksByAuthor({ db: context.db, ...input }),
          errors,
      );
  });

const searchBooksRoute = api
  .route({ method: 'GET' })
  .input(searchBooksInput)
  .output(getAllBooksOutput)
  .handler(async ({ context, input, errors }) => {
      return res(
          await searchBooks({ db: context.db, ...input }),
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
          await updateBook({ db: context.db, id, book }),
          errors,
      );
  });

const deleteBookRoute = api
  .route({ method: 'DELETE' })
  .input(deleteBookInput)
  .output(deleteBookOutput)
  .handler(async ({ context, input, errors }) => {
      return res(
          await deleteBook({ db: context.db, ...input }),
          errors,
      );
  });

// ==============================
// ROUTER CONFIG
// ==============================

export const router = {
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
          context: { env },
      });

      if (rpcResult.matched) {
          return rpcResult.response;
      }

      return new Response('Not Found', { status: 404 });
  },
} satisfies ExportedHandler<Cloudflare.Env>;

export type ApiRouter = typeof router;
```

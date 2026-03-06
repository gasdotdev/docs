# Postgres Pattern

## Example

## Database Functions (`gas/db/src/book.ts`)

```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';
import type { Client } from 'pg';

// ==============================
// SCHEMA
// ==============================

export const BookSchema = z.object({
  id: z.string(),
  title: z.string(),
  authorId: z.string(),
  publishedYear: z.number().nullable(),
});

// ==============================
// CREATE BOOK
// ==============================

export const createBookInput = BookSchema.omit({ id: true });

export type CreateBookOptions = {
  db: Client;
  book: z.infer<typeof createBookInput>;
};

export async function createBook({
  db,
  book,
}: CreateBookOptions): Promise<Result<z.infer<typeof BookSchema>>> {
  try {
    const bookId = nanoid();
    const result = await db.query(
      'INSERT INTO "Book" ("Id", "Title", "AuthorId", "PublishedYear") VALUES ($1, $2, $3, $4) RETURNING *',
      [bookId, book.title, book.authorId, book.publishedYear],
    );

    if (!result.rows[0]) {
      return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create book');
    }
    return ok(result.rows[0]);
  } catch (error: unknown) {
    const errorMessage =
      error instanceof Error ? error.message : 'Unknown error occurred';

    if (errorMessage.includes('23505') || errorMessage.includes('unique_violation')) {
      return err(ERR.CONFLICT, 'Book ID already exists');
    }

    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create book');
  }
}

// ==============================
// GET BOOK
// ==============================

export const getBookInput = z.object({ id: z.string() });

export async function getBook({
  db,
  id,
}: {
  db: Client;
  id: string;
}): Promise<Result<z.infer<typeof BookSchema>>> {
  try {
    const result = await db.query(
      'SELECT * FROM "Book" WHERE "Id" = $1',
      [id],
    );

    if (!result.rows[0]) {
      return err(ERR.NOT_FOUND, 'Book not found');
    }
    return ok(result.rows[0]);
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to get book');
  }
}

// ==============================
// GET ALL BOOKS
// ==============================

export const getAllBooksInput = z.object({}).optional();
export const getAllBooksOutput = z.array(BookSchema);

export async function getAllBooks({
  db,
}: {
  db: Client;
}): Promise<Result<z.infer<typeof getAllBooksOutput>>> {
  try {
    const result = await db.query('SELECT * FROM "Book"');
    return ok(result.rows);
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to get books');
  }
}

// ==============================
// GET BOOKS BY AUTHOR
// ==============================

export const getBooksByAuthorInput = z.object({ authorId: z.string() });

export async function getBooksByAuthor({
  db,
  authorId,
}: {
  db: Client;
  authorId: string;
}): Promise<Result<z.infer<typeof getAllBooksOutput>>> {
  try {
    const result = await db.query(
      'SELECT * FROM "Book" WHERE "AuthorId" = $1',
      [authorId],
    );
    return ok(result.rows);
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to get books by author');
  }
}

// ==============================
// SEARCH BOOKS
// ==============================

export const searchBooksInput = z.object({ query: z.string() });

export async function searchBooks({
  db,
  query,
}: {
  db: Client;
  query: string;
}): Promise<Result<z.infer<typeof getAllBooksOutput>>> {
  try {
    const result = await db.query(
      'SELECT * FROM "Book" WHERE "Title" ILIKE $1',
      [`%${query}%`],
    );
    return ok(result.rows);
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to search books');
  }
}

// ==============================
// UPDATE BOOK
// ==============================

export const updateBookInput = z.object({
  id: z.string(),
  title: z.string().optional(),
  authorId: z.string().optional(),
  publishedYear: z.number().nullable().optional(),
});

export async function updateBook({
  db,
  id,
  book,
}: {
  db: Client;
  id: string;
  book: Omit<z.infer<typeof updateBookInput>, 'id'>;
}): Promise<Result<z.infer<typeof BookSchema>>> {
  try {
    const fields: string[] = [];
    const values: unknown[] = [];
    let paramIndex = 1;

    if (book.title !== undefined) {
      fields.push(`"Title" = $${paramIndex++}`);
      values.push(book.title);
    }
    if (book.authorId !== undefined) {
      fields.push(`"AuthorId" = $${paramIndex++}`);
      values.push(book.authorId);
    }
    if (book.publishedYear !== undefined) {
      fields.push(`"PublishedYear" = $${paramIndex++}`);
      values.push(book.publishedYear);
    }

    if (fields.length === 0) {
      return err(ERR.BAD_REQUEST, 'No fields to update');
    }

    values.push(id);
    const result = await db.query(
      `UPDATE "Book" SET ${fields.join(', ')} WHERE "Id" = $${paramIndex} RETURNING *`,
      values,
    );

    if (!result.rows[0]) {
      return err(ERR.NOT_FOUND, 'Book not found');
    }
    return ok(result.rows[0]);
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update book');
  }
}

// ==============================
// DELETE BOOK
// ==============================

export const deleteBookInput = z.object({ id: z.string() });
export const deleteBookOutput = z.object({ success: z.boolean() });

export async function deleteBook({
  db,
  id,
}: {
  db: Client;
  id: string;
}): Promise<Result<z.infer<typeof deleteBookOutput>>> {
  try {
    const result = await db.query(
      'DELETE FROM "Book" WHERE "Id" = $1 RETURNING *',
      [id],
    );

    if (!result.rows[0]) {
      return err(ERR.NOT_FOUND, 'Book not found');
    }
    return ok({ success: true });
  } catch {
    return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete book');
  }
}
```

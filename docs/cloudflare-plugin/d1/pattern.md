# D1 Pattern

`gas/db/src/author.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const AuthorSchema = z.object({
	id: z.string(),
	name: z.string().nullable(),
});

// ==============================
// CREATE AUTHOR
// ==============================

export const createAuthorInput = AuthorSchema.omit({
	id: true,
});

export type CreateAuthorOptions = {
	db: D1Database;
	author: z.infer<typeof createAuthorInput>;
};

export async function createAuthor({
	db,
	author,
}: CreateAuthorOptions): Promise<Result<z.infer<typeof AuthorSchema>>> {
	try {
		const authorId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Author] (Id, Name) VALUES (?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(authorId, author.name)
			.first<z.infer<typeof AuthorSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create author');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Author ID already exists');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create author');
	}
}

// ==============================
// GET AUTHOR
// ==============================

export const getAuthorInput = AuthorSchema.pick({ id: true });

export type GetAuthorOptions = {
	db: D1Database;
	id: string;
};

export async function getAuthor({
	db,
	id,
}: GetAuthorOptions): Promise<Result<z.infer<typeof AuthorSchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Author] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof AuthorSchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Author not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get author: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL AUTHORS
// ==============================

export const getAllAuthorsInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllAuthorsOutput = z.object({
	authors: z.array(AuthorSchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllAuthorsOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllAuthors({
	db,
	page,
	limit,
}: GetAllAuthorsOptions): Promise<
	Result<z.infer<typeof getAllAuthorsOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Author]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count authors');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Author] ORDER BY Name ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof AuthorSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve authors');
		}

		return ok({
			authors: result.results || [],
			pagination: {
				page,
				limit,
				total,
				totalPages,
				hasNext: page < totalPages,
				hasPrev: page > 1,
			},
		});
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get authors: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE AUTHOR
// ==============================

export const updateAuthorInput = z.object({
	id: z.string(),
	name: z.string().nullable(),
});

export type UpdateAuthorOptions = {
	db: D1Database;
	id: string;
	author: Omit<z.infer<typeof updateAuthorInput>, 'id'>;
};

export async function updateAuthor({
	db,
	id,
	author,
}: UpdateAuthorOptions): Promise<Result<z.infer<typeof AuthorSchema>>> {
	try {
		const existingAuthor = await getAuthor({ db, id });
		if (existingAuthor.err) {
			return existingAuthor;
		}

		const stmt = db.prepare(
			'UPDATE [Author] SET Name = ? WHERE Id = ? RETURNING *',
		);

		const result = await stmt
			.bind(author.name, id)
			.first<z.infer<typeof AuthorSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update author');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update author');
	}
}

// ==============================
// DELETE AUTHOR
// ==============================

export const deleteAuthorInput = AuthorSchema.pick({ id: true });

export const deleteAuthorOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteAuthorOptions = {
	db: D1Database;
	id: string;
};

export async function deleteAuthor({
	db,
	id,
}: DeleteAuthorOptions): Promise<
	Result<z.infer<typeof deleteAuthorOutput>>
> {
	try {
		const existingAuthor = await getAuthor({ db, id });
		if (existingAuthor.err) {
			return existingAuthor;
		}

		const stmt = db.prepare('DELETE FROM [Author] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete author');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Author not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete author: ${errorMessage}`,
		);
	}
}
```

`gas/db/src/book.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const BookSchema = z.object({
	id: z.string(),
	authorId: z.string(),
	title: z.string().nullable(),
});

// ==============================
// CREATE BOOK
// ==============================

export const createBookInput = BookSchema.omit({
	id: true,
});

export type CreateBookOptions = {
	db: D1Database;
	book: z.infer<typeof createBookInput>;
};

export async function createBook({
	db,
	book,
}: CreateBookOptions): Promise<Result<z.infer<typeof BookSchema>>> {
	try {
		const bookId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Book] (Id, AuthorId, Title) VALUES (?, ?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(bookId, book.authorId, book.title)
			.first<z.infer<typeof BookSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create book');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Book ID already exists');
		}

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid author ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create book');
	}
}

// ==============================
// GET BOOK
// ==============================

export const getBookInput = BookSchema.pick({ id: true });

export type GetBookOptions = {
	db: D1Database;
	id: string;
};

export async function getBook({
	db,
	id,
}: GetBookOptions): Promise<Result<z.infer<typeof BookSchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Book] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof BookSchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Book not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get book: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL BOOKS
// ==============================

export const getAllBooksInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllBooksOutput = z.object({
	books: z.array(BookSchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllBooksOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllBooks({
	db,
	page,
	limit,
}: GetAllBooksOptions): Promise<Result<z.infer<typeof getAllBooksOutput>>> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Book]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count books');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Book] ORDER BY Title ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof BookSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve books');
		}

		return ok({
			books: result.results || [],
			pagination: {
				page,
				limit,
				total,
				totalPages,
				hasNext: page < totalPages,
				hasPrev: page > 1,
			},
		});
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get books: ${errorMessage}`,
		);
	}
}

// ==============================
// GET BOOKS BY AUTHOR
// ==============================

export const getBooksByAuthorInput = z.object({
	authorId: z.string(),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type GetBooksByAuthorOptions = {
	db: D1Database;
	authorId: string;
	page: number;
	limit: number;
};

export async function getBooksByAuthor({
	db,
	authorId,
	page,
	limit,
}: GetBooksByAuthorOptions): Promise<
	Result<z.infer<typeof getAllBooksOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Book] WHERE AuthorId = ?`;
		const countResult = await db
			.prepare(countQuery)
			.bind(authorId)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count books');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Book] WHERE AuthorId = ? ORDER BY Title ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(authorId, limit, offset)
			.all<z.infer<typeof BookSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve books');
		}

		return ok({
			books: result.results || [],
			pagination: {
				page,
				limit,
				total,
				totalPages,
				hasNext: page < totalPages,
				hasPrev: page > 1,
			},
		});
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get books by author: ${errorMessage}`,
		);
	}
}

// ==============================
// SEARCH BOOKS
// ==============================

export const searchBooksInput = z.object({
	query: z.string().min(1),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type SearchBooksOptions = {
	db: D1Database;
	query: string;
	page: number;
	limit: number;
};

export async function searchBooks({
	db,
	query,
	page,
	limit,
}: SearchBooksOptions): Promise<Result<z.infer<typeof getAllBooksOutput>>> {
	try {
		const offset = (page - 1) * limit;
		const searchPattern = `%${query}%`;

		const countQuery = `
			SELECT COUNT(*) as total FROM [Book]
			WHERE Title LIKE ?
		`;
		const countResult = await db
			.prepare(countQuery)
			.bind(searchPattern)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count books');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const searchQuery = `
			SELECT * FROM [Book]
			WHERE Title LIKE ?
			ORDER BY Title ASC
			LIMIT ? OFFSET ?
		`;
		const result = await db
			.prepare(searchQuery)
			.bind(searchPattern, limit, offset)
			.all<z.infer<typeof BookSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to search books');
		}

		return ok({
			books: result.results || [],
			pagination: {
				page,
				limit,
				total,
				totalPages,
				hasNext: page < totalPages,
				hasPrev: page > 1,
			},
		});
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to search books: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE BOOK
// ==============================

export const updateBookInput = z.object({
	id: z.string(),
	authorId: z.string(),
	title: z.string().nullable(),
});

export type UpdateBookOptions = {
	db: D1Database;
	id: string;
	book: Omit<z.infer<typeof updateBookInput>, 'id'>;
};

export async function updateBook({
	db,
	id,
	book,
}: UpdateBookOptions): Promise<Result<z.infer<typeof BookSchema>>> {
	try {
		const existingBook = await getBook({ db, id });
		if (existingBook.err) {
			return existingBook;
		}

		const stmt = db.prepare(
			'UPDATE [Book] SET AuthorId = ?, Title = ? WHERE Id = ? RETURNING *',
		);

		const result = await stmt
			.bind(book.authorId, book.title, id)
			.first<z.infer<typeof BookSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update book');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid author ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update book');
	}
}

// ==============================
// DELETE BOOK
// ==============================

export const deleteBookInput = BookSchema.pick({ id: true });

export const deleteBookOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteBookOptions = {
	db: D1Database;
	id: string;
};

export async function deleteBook({
	db,
	id,
}: DeleteBookOptions): Promise<Result<z.infer<typeof deleteBookOutput>>> {
	try {
		const existingBook = await getBook({ db, id });
		if (existingBook.err) {
			return existingBook;
		}

		const stmt = db.prepare('DELETE FROM [Book] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete book');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Book not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete book: ${errorMessage}`,
		);
	}
}
```

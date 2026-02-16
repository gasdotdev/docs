# D1

D1 databases are accessed with functions and SQL instead of ORMs.

The functions can then be used throughout a project, including seeding and testing.

`packages/core/src/index.ts`:
```ts
import { z } from 'zod';
import type { ORPCErrorConstructorMap } from '@orpc/server';

export const ERRS = {
	BAD_REQUEST: {},
	INTERNAL_SERVER_ERROR: {},
	CONFLICT: {},
	NOT_FOUND: {},
	UNAUTHORIZED: {},
	RATE_LIMITED: {
		data: z.object({
			retryAfter: z.number(),
		}),
	},
} as const;

export type Err = keyof typeof ERRS;

export const ERR = Object.keys(ERRS).reduce(
	(acc, key) => {
		acc[key as keyof typeof ERRS] = key;
		return acc;
	},
	{} as Record<keyof typeof ERRS, string>,
) as {
	readonly [K in keyof typeof ERRS]: K;
};

export type Result<T> =
	| (T & { err?: undefined })
	| {
			err: Err;
			message: string;
	  };

export function ok<T>(data: T): Result<T> {
	return data as T & { err?: undefined };
}

export function err(err: Err, message: string): Result<never> {
	return {
		err,
		message,
	};
}

export function res<TData>(
	result: Result<TData>,
	errors: ORPCErrorConstructorMap<typeof ERRS>,
) {
	if (result.err) {
		switch (result.err) {
			case ERR.BAD_REQUEST:
			case ERR.INTERNAL_SERVER_ERROR:
			case ERR.CONFLICT:
			case ERR.NOT_FOUND:
			case ERR.UNAUTHORIZED:
				throw errors[result.err]({ message: result.message });
			case ERR.RATE_LIMITED:
				throw errors[result.err]({
					message: result.message,
					data: { retryAfter: 60 },
				});
			default:
				throw errors.INTERNAL_SERVER_ERROR({ message: 'Unknown error' });
		}
	}
	return result as TData;
}
```

`gas/db/src/category.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const CategorySchema = z.object({
	id: z.string(),
	name: z.string().nullable(),
	description: z.string().nullable(),
});

// ==============================
// CREATE CATEGORY
// ==============================

export const createCategoryInput = CategorySchema.omit({
	id: true,
});

export type CreateCategoryOptions = {
	db: D1Database;
	category: z.infer<typeof createCategoryInput>;
};

export async function createCategory({
	db,
	category,
}: CreateCategoryOptions): Promise<Result<z.infer<typeof CategorySchema>>> {
	try {
		const categoryId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Category] (Id, Name, Description) VALUES (?, ?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(categoryId, category.name, category.description)
			.first<z.infer<typeof CategorySchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create category');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Category ID already exists');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create category');
	}
}

// ==============================
// GET CATEGORY
// ==============================

export const getCategoryInput = CategorySchema.pick({ id: true });

export type GetCategoryOptions = {
	db: D1Database;
	id: string;
};

export async function getCategory({
	db,
	id,
}: GetCategoryOptions): Promise<Result<z.infer<typeof CategorySchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Category] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof CategorySchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Category not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get category: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL CATEGORIES
// ==============================

export const getAllCategoriesInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllCategoriesOutput = z.object({
	categories: z.array(CategorySchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllCategoriesOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllCategories({
	db,
	page,
	limit,
}: GetAllCategoriesOptions): Promise<
	Result<z.infer<typeof getAllCategoriesOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Category]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count categories');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Category] ORDER BY Name ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof CategorySchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve categories');
		}

		return ok({
			categories: result.results || [],
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
			`Unable to get categories: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE CATEGORY
// ==============================

export const updateCategoryInput = z.object({
	id: z.string(),
	name: z.string().nullable(),
	description: z.string().nullable(),
});

export type UpdateCategoryOptions = {
	db: D1Database;
	id: string;
	category: Omit<z.infer<typeof updateCategoryInput>, 'id'>;
};

export async function updateCategory({
	db,
	id,
	category,
}: UpdateCategoryOptions): Promise<Result<z.infer<typeof CategorySchema>>> {
	try {
		const existingCategory = await getCategory({ db, id });
		if (existingCategory.err) {
			return existingCategory;
		}

		const stmt = db.prepare(
			'UPDATE [Category] SET Name = ?, Description = ? WHERE Id = ? RETURNING *',
		);

		const result = await stmt
			.bind(category.name, category.description, id)
			.first<z.infer<typeof CategorySchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update category');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update category');
	}
}

// ==============================
// DELETE CATEGORY
// ==============================

export const deleteCategoryInput = CategorySchema.pick({ id: true });

export const deleteCategoryOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteCategoryOptions = {
	db: D1Database;
	id: string;
};

export async function deleteCategory({
	db,
	id,
}: DeleteCategoryOptions): Promise<
	Result<z.infer<typeof deleteCategoryOutput>>
> {
	try {
		const existingCategory = await getCategory({ db, id });
		if (existingCategory.err) {
			return existingCategory;
		}

		const stmt = db.prepare('DELETE FROM [Category] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete category');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Category not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete category: ${errorMessage}`,
		);
	}
}
```

`gas/db/src/customer.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const CustomerSchema = z.object({
	id: z.string(),
	firstName: z.string().nullable(),
	lastName: z.string().nullable(),
	address: z.string().nullable(),
	city: z.string().nullable(),
	state: z.string().nullable(),
	postalCode: z.string().nullable(),
	country: z.string().nullable(),
});

// ==============================
// CREATE CUSTOMER
// ==============================

export const createCustomerInput = CustomerSchema.omit({
	id: true,
});

export type CreateCustomerOptions = {
	db: D1Database;
	customer: z.infer<typeof createCustomerInput>;
};

export async function createCustomer({
	db,
	customer,
}: CreateCustomerOptions): Promise<Result<z.infer<typeof CustomerSchema>>> {
	try {
		const customerId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Customer] (Id, FirstName, LastName, Address, City, State, PostalCode, Country) VALUES (?, ?, ?, ?, ?, ?, ?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(
				customerId,
				customer.firstName,
				customer.lastName,
				customer.address,
				customer.city,
				customer.state,
				customer.postalCode,
				customer.country,
			)
			.first<z.infer<typeof CustomerSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create customer');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Customer ID already exists');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create customer');
	}
}

// ==============================
// GET CUSTOMER
// ==============================

export const getCustomerInput = CustomerSchema.pick({ id: true });

export type GetCustomerOptions = {
	db: D1Database;
	id: string;
};

export async function getCustomer({
	db,
	id,
}: GetCustomerOptions): Promise<Result<z.infer<typeof CustomerSchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Customer] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof CustomerSchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Customer not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get customer: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL CUSTOMERS
// ==============================

export const getAllCustomersInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllCustomersOutput = z.object({
	customers: z.array(CustomerSchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllCustomersOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllCustomers({
	db,
	page,
	limit,
}: GetAllCustomersOptions): Promise<
	Result<z.infer<typeof getAllCustomersOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Customer]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count customers');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Customer] ORDER BY LastName ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof CustomerSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve customers');
		}

		return ok({
			customers: result.results || [],
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
			`Unable to get customers: ${errorMessage}`,
		);
	}
}

// ==============================
// SEARCH CUSTOMERS
// ==============================

export const searchCustomersInput = z.object({
	query: z.string().min(1),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type SearchCustomersOptions = {
	db: D1Database;
	query: string;
	page: number;
	limit: number;
};

export async function searchCustomers({
	db,
	query,
	page,
	limit,
}: SearchCustomersOptions): Promise<
	Result<z.infer<typeof getAllCustomersOutput>>
> {
	try {
		const offset = (page - 1) * limit;
		const searchPattern = `%${query}%`;

		const countQuery = `
			SELECT COUNT(*) as total FROM [Customer]
			WHERE FirstName LIKE ? OR LastName LIKE ? OR City LIKE ? OR Country LIKE ?
		`;
		const countResult = await db
			.prepare(countQuery)
			.bind(searchPattern, searchPattern, searchPattern, searchPattern)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count customers');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const searchQuery = `
			SELECT * FROM [Customer]
			WHERE FirstName LIKE ? OR LastName LIKE ? OR City LIKE ? OR Country LIKE ?
			ORDER BY LastName ASC
			LIMIT ? OFFSET ?
		`;
		const result = await db
			.prepare(searchQuery)
			.bind(
				searchPattern,
				searchPattern,
				searchPattern,
				searchPattern,
				limit,
				offset,
			)
			.all<z.infer<typeof CustomerSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to search customers');
		}

		return ok({
			customers: result.results || [],
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
			`Unable to search customers: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE CUSTOMER
// ==============================

export const updateCustomerInput = z.object({
	id: z.string(),
	firstName: z.string().nullable(),
	lastName: z.string().nullable(),
	address: z.string().nullable(),
	city: z.string().nullable(),
	state: z.string().nullable(),
	postalCode: z.string().nullable(),
	country: z.string().nullable(),
});

export type UpdateCustomerOptions = {
	db: D1Database;
	id: string;
	customer: Omit<z.infer<typeof updateCustomerInput>, 'id'>;
};

export async function updateCustomer({
	db,
	id,
	customer,
}: UpdateCustomerOptions): Promise<Result<z.infer<typeof CustomerSchema>>> {
	try {
		const existingCustomer = await getCustomer({ db, id });
		if (existingCustomer.err) {
			return existingCustomer;
		}

		const stmt = db.prepare(`
			UPDATE [Customer] SET
				FirstName = ?,
				LastName = ?,
				Address = ?,
				City = ?,
				State = ?,
				PostalCode = ?,
				Country = ?
			WHERE Id = ?
			RETURNING *
		`);

		const result = await stmt
			.bind(
				customer.firstName,
				customer.lastName,
				customer.address,
				customer.city,
				customer.state,
				customer.postalCode,
				customer.country,
				id,
			)
			.first<z.infer<typeof CustomerSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update customer');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update customer');
	}
}

// ==============================
// DELETE CUSTOMER
// ==============================

export const deleteCustomerInput = CustomerSchema.pick({ id: true });

export const deleteCustomerOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteCustomerOptions = {
	db: D1Database;
	id: string;
};

export async function deleteCustomer({
	db,
	id,
}: DeleteCustomerOptions): Promise<
	Result<z.infer<typeof deleteCustomerOutput>>
> {
	try {
		const existingCustomer = await getCustomer({ db, id });
		if (existingCustomer.err) {
			return existingCustomer;
		}

		const stmt = db.prepare('DELETE FROM [Customer] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete customer');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Customer not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete customer: ${errorMessage}`,
		);
	}
}
```

`gas/db/src/product.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const ProductSchema = z.object({
	id: z.string(),
	name: z.string().nullable(),
	categoryId: z.string(),
	price: z.number(),
});

// ==============================
// CREATE PRODUCT
// ==============================

export const createProductInput = ProductSchema.omit({
	id: true,
});

export type CreateProductOptions = {
	db: D1Database;
	product: z.infer<typeof createProductInput>;
};

export async function createProduct({
	db,
	product,
}: CreateProductOptions): Promise<Result<z.infer<typeof ProductSchema>>> {
	try {
		const productId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Product] (Id, Name, CategoryId, Price) VALUES (?, ?, ?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(productId, product.name, product.categoryId, product.price)
			.first<z.infer<typeof ProductSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create product');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Product ID already exists');
		}

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid category ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create product');
	}
}

// ==============================
// GET PRODUCT
// ==============================

export const getProductInput = ProductSchema.pick({ id: true });

export type GetProductOptions = {
	db: D1Database;
	id: string;
};

export async function getProduct({
	db,
	id,
}: GetProductOptions): Promise<Result<z.infer<typeof ProductSchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Product] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof ProductSchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Product not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get product: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL PRODUCTS
// ==============================

export const getAllProductsInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllProductsOutput = z.object({
	products: z.array(ProductSchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllProductsOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllProducts({
	db,
	page,
	limit,
}: GetAllProductsOptions): Promise<
	Result<z.infer<typeof getAllProductsOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Product]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count products');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Product] ORDER BY Name ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof ProductSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve products');
		}

		return ok({
			products: result.results || [],
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
			`Unable to get products: ${errorMessage}`,
		);
	}
}

// ==============================
// GET PRODUCTS BY CATEGORY
// ==============================

export const getProductsByCategoryInput = z.object({
	categoryId: z.string(),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type GetProductsByCategoryOptions = {
	db: D1Database;
	categoryId: string;
	page: number;
	limit: number;
};

export async function getProductsByCategory({
	db,
	categoryId,
	page,
	limit,
}: GetProductsByCategoryOptions): Promise<
	Result<z.infer<typeof getAllProductsOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Product] WHERE CategoryId = ?`;
		const countResult = await db
			.prepare(countQuery)
			.bind(categoryId)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count products');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Product] WHERE CategoryId = ? ORDER BY Name ASC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(categoryId, limit, offset)
			.all<z.infer<typeof ProductSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve products');
		}

		return ok({
			products: result.results || [],
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
			`Unable to get products by category: ${errorMessage}`,
		);
	}
}

// ==============================
// SEARCH PRODUCTS
// ==============================

export const searchProductsInput = z.object({
	query: z.string().min(1),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type SearchProductsOptions = {
	db: D1Database;
	query: string;
	page: number;
	limit: number;
};

export async function searchProducts({
	db,
	query,
	page,
	limit,
}: SearchProductsOptions): Promise<
	Result<z.infer<typeof getAllProductsOutput>>
> {
	try {
		const offset = (page - 1) * limit;
		const searchPattern = `%${query}%`;

		const countQuery = `
			SELECT COUNT(*) as total FROM [Product]
			WHERE Name LIKE ?
		`;
		const countResult = await db
			.prepare(countQuery)
			.bind(searchPattern)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count products');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const searchQuery = `
			SELECT * FROM [Product]
			WHERE Name LIKE ?
			ORDER BY Name ASC
			LIMIT ? OFFSET ?
		`;
		const result = await db
			.prepare(searchQuery)
			.bind(searchPattern, limit, offset)
			.all<z.infer<typeof ProductSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to search products');
		}

		return ok({
			products: result.results || [],
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
			`Unable to search products: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE PRODUCT
// ==============================

export const updateProductInput = z.object({
	id: z.string(),
	name: z.string().nullable(),
	categoryId: z.string(),
	price: z.number(),
});

export type UpdateProductOptions = {
	db: D1Database;
	id: string;
	product: Omit<z.infer<typeof updateProductInput>, 'id'>;
};

export async function updateProduct({
	db,
	id,
	product,
}: UpdateProductOptions): Promise<Result<z.infer<typeof ProductSchema>>> {
	try {
		const existingProduct = await getProduct({ db, id });
		if (existingProduct.err) {
			return existingProduct;
		}

		const stmt = db.prepare(
			'UPDATE [Product] SET Name = ?, CategoryId = ?, Price = ? WHERE Id = ? RETURNING *',
		);

		const result = await stmt
			.bind(product.name, product.categoryId, product.price, id)
			.first<z.infer<typeof ProductSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update product');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid category ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update product');
	}
}

// ==============================
// DELETE PRODUCT
// ==============================

export const deleteProductInput = ProductSchema.pick({ id: true });

export const deleteProductOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteProductOptions = {
	db: D1Database;
	id: string;
};

export async function deleteProduct({
	db,
	id,
}: DeleteProductOptions): Promise<Result<z.infer<typeof deleteProductOutput>>> {
	try {
		const existingProduct = await getProduct({ db, id });
		if (existingProduct.err) {
			return existingProduct;
		}

		const stmt = db.prepare('DELETE FROM [Product] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete product');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Product not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete product: ${errorMessage}`,
		);
	}
}
```

`gas/db/src/order.ts`:
```ts
import { nanoid } from 'nanoid';
import { z } from 'zod';
import { ERR, ok, err } from 'core';
import type { Result } from 'core';

// ==============================
// SCHEMA
// ==============================

export const OrderSchema = z.object({
	id: z.string(),
	customerId: z.string().nullable(),
	date: z.number(),
	cost: z.number(),
});

// ==============================
// CREATE ORDER
// ==============================

export const createOrderInput = OrderSchema.omit({
	id: true,
});

export type CreateOrderOptions = {
	db: D1Database;
	order: z.infer<typeof createOrderInput>;
};

export async function createOrder({
	db,
	order,
}: CreateOrderOptions): Promise<Result<z.infer<typeof OrderSchema>>> {
	try {
		const orderId = nanoid();
		const stmt = db.prepare(
			'INSERT INTO [Order] (Id, CustomerId, Date, Cost) VALUES (?, ?, ?, ?) RETURNING *',
		);

		const result = await stmt
			.bind(orderId, order.customerId, order.date, order.cost)
			.first<z.infer<typeof OrderSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create order');
		}
		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('UNIQUE constraint failed')) {
			return err(ERR.CONFLICT, 'Order ID already exists');
		}

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid customer ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to create order');
	}
}

// ==============================
// GET ORDER
// ==============================

export const getOrderInput = OrderSchema.pick({ id: true });

export type GetOrderOptions = {
	db: D1Database;
	id: string;
};

export async function getOrder({
	db,
	id,
}: GetOrderOptions): Promise<Result<z.infer<typeof OrderSchema>>> {
	try {
		const stmt = db.prepare('SELECT * FROM [Order] WHERE Id = ?');
		const result = await stmt.bind(id).first<z.infer<typeof OrderSchema>>();

		if (!result) {
			return err(ERR.NOT_FOUND, 'Order not found');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to get order: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ALL ORDERS
// ==============================

export const getAllOrdersInput = z.object({
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export const getAllOrdersOutput = z.object({
	orders: z.array(OrderSchema),
	pagination: z.object({
		page: z.number(),
		limit: z.number(),
		total: z.number(),
		totalPages: z.number(),
		hasNext: z.boolean(),
		hasPrev: z.boolean(),
	}),
});

export type GetAllOrdersOptions = {
	db: D1Database;
	page: number;
	limit: number;
};

export async function getAllOrders({
	db,
	page,
	limit,
}: GetAllOrdersOptions): Promise<Result<z.infer<typeof getAllOrdersOutput>>> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Order]`;
		const countResult = await db.prepare(countQuery).first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count orders');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Order] ORDER BY Date DESC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(limit, offset)
			.all<z.infer<typeof OrderSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve orders');
		}

		return ok({
			orders: result.results || [],
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
			`Unable to get orders: ${errorMessage}`,
		);
	}
}

// ==============================
// GET ORDERS BY CUSTOMER
// ==============================

export const getOrdersByCustomerInput = z.object({
	customerId: z.string(),
	page: z.number().int().min(1).default(1),
	limit: z.number().int().min(1).max(100).default(10),
});

export type GetOrdersByCustomerOptions = {
	db: D1Database;
	customerId: string;
	page: number;
	limit: number;
};

export async function getOrdersByCustomer({
	db,
	customerId,
	page,
	limit,
}: GetOrdersByCustomerOptions): Promise<
	Result<z.infer<typeof getAllOrdersOutput>>
> {
	try {
		const offset = (page - 1) * limit;

		const countQuery = `SELECT COUNT(*) as total FROM [Order] WHERE CustomerId = ?`;
		const countResult = await db
			.prepare(countQuery)
			.bind(customerId)
			.first<{ total: number }>();

		if (!countResult) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to count orders');
		}

		const total = countResult.total;
		const totalPages = Math.ceil(total / limit);

		const query = `SELECT * FROM [Order] WHERE CustomerId = ? ORDER BY Date DESC LIMIT ? OFFSET ?`;
		const result = await db
			.prepare(query)
			.bind(customerId, limit, offset)
			.all<z.infer<typeof OrderSchema>>();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Failed to retrieve orders');
		}

		return ok({
			orders: result.results || [],
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
			`Unable to get orders by customer: ${errorMessage}`,
		);
	}
}

// ==============================
// UPDATE ORDER
// ==============================

export const updateOrderInput = z.object({
	id: z.string(),
	customerId: z.string().nullable(),
	date: z.number(),
	cost: z.number(),
});

export type UpdateOrderOptions = {
	db: D1Database;
	id: string;
	order: Omit<z.infer<typeof updateOrderInput>, 'id'>;
};

export async function updateOrder({
	db,
	id,
	order,
}: UpdateOrderOptions): Promise<Result<z.infer<typeof OrderSchema>>> {
	try {
		const existingOrder = await getOrder({ db, id });
		if (existingOrder.err) {
			return existingOrder;
		}

		const stmt = db.prepare(`
			UPDATE [Order] SET
				CustomerId = ?,
				Date = ?,
				Cost = ?
			WHERE Id = ?
			RETURNING *
		`);

		const result = await stmt
			.bind(order.customerId, order.date, order.cost, id)
			.first<z.infer<typeof OrderSchema>>();

		if (!result) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update order');
		}

		return ok(result);
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';

		if (errorMessage.includes('FOREIGN KEY constraint failed')) {
			return err(ERR.BAD_REQUEST, 'Invalid customer ID');
		}

		return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to update order');
	}
}

// ==============================
// DELETE ORDER
// ==============================

export const deleteOrderInput = OrderSchema.pick({ id: true });

export const deleteOrderOutput = z.object({
	id: z.string(),
	changes: z.number(),
});

export type DeleteOrderOptions = {
	db: D1Database;
	id: string;
};

export async function deleteOrder({
	db,
	id,
}: DeleteOrderOptions): Promise<Result<z.infer<typeof deleteOrderOutput>>> {
	try {
		const existingOrder = await getOrder({ db, id });
		if (existingOrder.err) {
			return existingOrder;
		}

		const stmt = db.prepare('DELETE FROM [Order] WHERE Id = ?');
		const result = await stmt.bind(id).run();

		if (!result.success) {
			return err(ERR.INTERNAL_SERVER_ERROR, 'Unable to delete order');
		}

		if (result.meta.changes === 0) {
			return err(ERR.NOT_FOUND, 'Order not found');
		}

		return ok({ id, changes: result.meta.changes });
	} catch (error: unknown) {
		const errorMessage =
			error instanceof Error ? error.message : 'Unknown error occurred';
		return err(
			ERR.INTERNAL_SERVER_ERROR,
			`Unable to delete order: ${errorMessage}`,
		);
	}
}
```

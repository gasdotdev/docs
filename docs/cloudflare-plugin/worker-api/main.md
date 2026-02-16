# Worker API

Worker APIs use oRPC.

The API itself should be "dumb".

All core logic is pushed into [reusable database functions](../d1/main.md).

oRPC docs: https://orpc.dev/docs/getting-started

oRPC OpenAPI docs: https://orpc.dev/docs/openapi/getting-started

oRPC llms.txt: https://orpc.dev/llms.txt

## Example

`gas/api/src/index.ts`:
```ts
import { os } from '@orpc/server';
import { RPCHandler } from '@orpc/server/fetch';
import {
	createCategory,
	createCategoryInput,
	getCategory,
	getCategoryInput,
	getAllCategories,
	getAllCategoriesInput,
	getAllCategoriesOutput,
	updateCategory,
	updateCategoryInput,
	deleteCategory,
	deleteCategoryInput,
	deleteCategoryOutput,
	CategorySchema,
} from '../../db/src/category.ts';
import {
	createCustomer,
	createCustomerInput,
	getCustomer,
	getCustomerInput,
	getAllCustomers,
	getAllCustomersInput,
	getAllCustomersOutput,
	searchCustomers,
	searchCustomersInput,
	updateCustomer,
	updateCustomerInput,
	deleteCustomer,
	deleteCustomerInput,
	deleteCustomerOutput,
	CustomerSchema,
} from '../../db/src/customer.ts';
import {
	createProduct,
	createProductInput,
	getProduct,
	getProductInput,
	getAllProducts,
	getAllProductsInput,
	getAllProductsOutput,
	getProductsByCategory,
	getProductsByCategoryInput,
	searchProducts,
	searchProductsInput,
	updateProduct,
	updateProductInput,
	deleteProduct,
	deleteProductInput,
	deleteProductOutput,
	ProductSchema,
} from '../../db/src/product.ts';
import {
	createOrder,
	createOrderInput,
	getOrder,
	getOrderInput,
	getAllOrders,
	getAllOrdersInput,
	getAllOrdersOutput,
	getOrdersByCustomer,
	getOrdersByCustomerInput,
	updateOrder,
	updateOrderInput,
	deleteOrder,
	deleteOrderInput,
	deleteOrderOutput,
	OrderSchema,
} from '../../db/src/order.ts';
import { res, ERRS } from 'core';

const base = os.$context<{ env: Cloudflare.Env }>().errors(ERRS);

const rateLimit = base.middleware(async ({ next, errors }) => {
	return next();
});

const api = base.use(rateLimit);

// ==============================
// CATEGORY ROUTES
// ==============================

const createCategoryRoute = api
	.route({ method: 'POST' })
	.input(createCategoryInput)
	.output(CategorySchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createCategory({
				db: context.env['root-db'],
				category: { ...input },
			}),
			errors,
		);
	});

const getCategoryRoute = api
	.route({ method: 'GET' })
	.input(getCategoryInput)
	.output(CategorySchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getCategory({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllCategoriesRoute = api
	.route({ method: 'GET' })
	.input(getAllCategoriesInput)
	.output(getAllCategoriesOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllCategories({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateCategoryRoute = api
	.route({ method: 'PUT' })
	.input(updateCategoryInput)
	.output(CategorySchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...category } = input;
		return res(
			await updateCategory({
				db: context.env['root-db'],
				id,
				category,
			}),
			errors,
		);
	});

const deleteCategoryRoute = api
	.route({ method: 'DELETE' })
	.input(deleteCategoryInput)
	.output(deleteCategoryOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteCategory({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

// ==============================
// CUSTOMER ROUTES
// ==============================

const createCustomerRoute = api
	.route({ method: 'POST' })
	.input(createCustomerInput)
	.output(CustomerSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createCustomer({
				db: context.env['root-db'],
				customer: { ...input },
			}),
			errors,
		);
	});

const getCustomerRoute = api
	.route({ method: 'GET' })
	.input(getCustomerInput)
	.output(CustomerSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getCustomer({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllCustomersRoute = api
	.route({ method: 'GET' })
	.input(getAllCustomersInput)
	.output(getAllCustomersOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllCustomers({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const searchCustomersRoute = api
	.route({ method: 'GET' })
	.input(searchCustomersInput)
	.output(getAllCustomersOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await searchCustomers({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateCustomerRoute = api
	.route({ method: 'PUT' })
	.input(updateCustomerInput)
	.output(CustomerSchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...customer } = input;
		return res(
			await updateCustomer({
				db: context.env['root-db'],
				id,
				customer,
			}),
			errors,
		);
	});

const deleteCustomerRoute = api
	.route({ method: 'DELETE' })
	.input(deleteCustomerInput)
	.output(deleteCustomerOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteCustomer({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

// ==============================
// PRODUCT ROUTES
// ==============================

const createProductRoute = api
	.route({ method: 'POST' })
	.input(createProductInput)
	.output(ProductSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createProduct({
				db: context.env['root-db'],
				product: { ...input },
			}),
			errors,
		);
	});

const getProductRoute = api
	.route({ method: 'GET' })
	.input(getProductInput)
	.output(ProductSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getProduct({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllProductsRoute = api
	.route({ method: 'GET' })
	.input(getAllProductsInput)
	.output(getAllProductsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllProducts({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getProductsByCategoryRoute = api
	.route({ method: 'GET' })
	.input(getProductsByCategoryInput)
	.output(getAllProductsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getProductsByCategory({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const searchProductsRoute = api
	.route({ method: 'GET' })
	.input(searchProductsInput)
	.output(getAllProductsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await searchProducts({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateProductRoute = api
	.route({ method: 'PUT' })
	.input(updateProductInput)
	.output(ProductSchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...product } = input;
		return res(
			await updateProduct({
				db: context.env['root-db'],
				id,
				product,
			}),
			errors,
		);
	});

const deleteProductRoute = api
	.route({ method: 'DELETE' })
	.input(deleteProductInput)
	.output(deleteProductOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteProduct({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

// ==============================
// ORDER ROUTES
// ==============================

const createOrderRoute = api
	.route({ method: 'POST' })
	.input(createOrderInput)
	.output(OrderSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createOrder({
				db: context.env['root-db'],
				order: { ...input },
			}),
			errors,
		);
	});

const getOrderRoute = api
	.route({ method: 'GET' })
	.input(getOrderInput)
	.output(OrderSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getOrder({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getAllOrdersRoute = api
	.route({ method: 'GET' })
	.input(getAllOrdersInput)
	.output(getAllOrdersOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllOrders({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const getOrdersByCustomerRoute = api
	.route({ method: 'GET' })
	.input(getOrdersByCustomerInput)
	.output(getAllOrdersOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getOrdersByCustomer({
				db: context.env['root-db'],
				...input,
			}),
			errors,
		);
	});

const updateOrderRoute = api
	.route({ method: 'PUT' })
	.input(updateOrderInput)
	.output(OrderSchema)
	.handler(async ({ context, input, errors }) => {
		const { id, ...order } = input;
		return res(
			await updateOrder({
				db: context.env['root-db'],
				id,
				order,
			}),
			errors,
		);
	});

const deleteOrderRoute = api
	.route({ method: 'DELETE' })
	.input(deleteOrderInput)
	.output(deleteOrderOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteOrder({
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
	categories: {
		create: createCategoryRoute,
		get: getCategoryRoute,
		getAll: getAllCategoriesRoute,
		update: updateCategoryRoute,
		delete: deleteCategoryRoute,
	},
	customers: {
		create: createCustomerRoute,
		get: getCustomerRoute,
		getAll: getAllCustomersRoute,
		search: searchCustomersRoute,
		update: updateCustomerRoute,
		delete: deleteCustomerRoute,
	},
	products: {
		create: createProductRoute,
		get: getProductRoute,
		getAll: getAllProductsRoute,
		getByCategory: getProductsByCategoryRoute,
		search: searchProductsRoute,
		update: updateProductRoute,
		delete: deleteProductRoute,
	},
	orders: {
		create: createOrderRoute,
		get: getOrderRoute,
		getAll: getAllOrdersRoute,
		getByCustomer: getOrdersByCustomerRoute,
		update: updateOrderRoute,
		delete: deleteOrderRoute,
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

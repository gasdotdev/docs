# Worker API

## ORPC

Worker API's use oRPC.

oRPC docs: https://orpc.dev/docs/getting-started

oRPC OpenAPI docs: https://orpc.dev/docs/openapi/getting-started

oRPC llms.txt: https://orpc.dev/llms.txt

## ORPC Example

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
import {
	createOrderDetail,
	createOrderDetailInput,
	getOrderDetail,
	getOrderDetailInput,
	getOrderDetailsByOrder,
	getOrderDetailsByOrderInput,
	getAllOrderDetails,
	getAllOrderDetailsInput,
	getAllOrderDetailsOutput,
	updateOrderDetail,
	updateOrderDetailInput,
	deleteOrderDetail,
	deleteOrderDetailInput,
	deleteOrderDetailOutput,
	OrderDetailSchema,
} from '../../db/src/order-detail.ts';
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
				id: input.id,
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
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const updateCategoryRoute = api
	.route({ method: 'PUT' })
	.input(updateCategoryInput)
	.output(CategorySchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await updateCategory({
				db: context.env['root-db'],
				id: input.id,
				category: {
					categoryName: input.categoryName,
					description: input.description,
				},
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
				id: input.id,
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
				id: input.id,
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
				page: input.page,
				limit: input.limit,
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
				query: input.query,
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const updateCustomerRoute = api
	.route({ method: 'PUT' })
	.input(updateCustomerInput)
	.output(CustomerSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await updateCustomer({
				db: context.env['root-db'],
				id: input.id,
				customer: {
					companyName: input.companyName,
					contactName: input.contactName,
					contactTitle: input.contactTitle,
					address: input.address,
					city: input.city,
					region: input.region,
					postalCode: input.postalCode,
					country: input.country,
					phone: input.phone,
					fax: input.fax,
				},
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
				id: input.id,
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
				id: input.id,
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
				page: input.page,
				limit: input.limit,
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
				customerId: input.customerId,
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const updateOrderRoute = api
	.route({ method: 'PUT' })
	.input(updateOrderInput)
	.output(OrderSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await updateOrder({
				db: context.env['root-db'],
				id: input.id,
				order: {
					customerId: input.customerId,
					orderDate: input.orderDate,
					requiredDate: input.requiredDate,
					shippedDate: input.shippedDate,
					freight: input.freight,
					shipName: input.shipName,
					shipAddress: input.shipAddress,
					shipCity: input.shipCity,
					shipRegion: input.shipRegion,
					shipPostalCode: input.shipPostalCode,
					shipCountry: input.shipCountry,
				},
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
				id: input.id,
			}),
			errors,
		);
	});

// ==============================
// ORDER DETAIL ROUTES
// ==============================

const createOrderDetailRoute = api
	.route({ method: 'POST' })
	.input(createOrderDetailInput)
	.output(OrderDetailSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await createOrderDetail({
				db: context.env['root-db'],
				orderDetail: { ...input },
			}),
			errors,
		);
	});

const getOrderDetailRoute = api
	.route({ method: 'GET' })
	.input(getOrderDetailInput)
	.output(OrderDetailSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getOrderDetail({
				db: context.env['root-db'],
				id: input.id,
			}),
			errors,
		);
	});

const getOrderDetailsByOrderRoute = api
	.route({ method: 'GET' })
	.input(getOrderDetailsByOrderInput)
	.output(getAllOrderDetailsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getOrderDetailsByOrder({
				db: context.env['root-db'],
				orderId: input.orderId,
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const getAllOrderDetailsRoute = api
	.route({ method: 'GET' })
	.input(getAllOrderDetailsInput)
	.output(getAllOrderDetailsOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await getAllOrderDetails({
				db: context.env['root-db'],
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const updateOrderDetailRoute = api
	.route({ method: 'PUT' })
	.input(updateOrderDetailInput)
	.output(OrderDetailSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await updateOrderDetail({
				db: context.env['root-db'],
				id: input.id,
				orderDetail: {
					orderId: input.orderId,
					productId: input.productId,
					unitPrice: input.unitPrice,
					quantity: input.quantity,
					discount: input.discount,
				},
			}),
			errors,
		);
	});

const deleteOrderDetailRoute = api
	.route({ method: 'DELETE' })
	.input(deleteOrderDetailInput)
	.output(deleteOrderDetailOutput)
	.handler(async ({ context, input, errors }) => {
		return res(
			await deleteOrderDetail({
				db: context.env['root-db'],
				id: input.id,
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
				id: input.id,
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
				page: input.page,
				limit: input.limit,
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
				categoryId: input.categoryId,
				page: input.page,
				limit: input.limit,
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
				query: input.query,
				page: input.page,
				limit: input.limit,
			}),
			errors,
		);
	});

const updateProductRoute = api
	.route({ method: 'PUT' })
	.input(updateProductInput)
	.output(ProductSchema)
	.handler(async ({ context, input, errors }) => {
		return res(
			await updateProduct({
				db: context.env['root-db'],
				id: input.id,
				product: {
					productName: input.productName,
					categoryId: input.categoryId,
					quantityPerUnit: input.quantityPerUnit,
					unitPrice: input.unitPrice,
					unitsInStock: input.unitsInStock,
					unitsOnOrder: input.unitsOnOrder,
					reorderLevel: input.reorderLevel,
					discontinued: input.discontinued,
				},
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
				id: input.id,
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
	orders: {
		create: createOrderRoute,
		get: getOrderRoute,
		getAll: getAllOrdersRoute,
		getByCustomer: getOrdersByCustomerRoute,
		update: updateOrderRoute,
		delete: deleteOrderRoute,
	},
	orderDetails: {
		create: createOrderDetailRoute,
		get: getOrderDetailRoute,
		getByOrder: getOrderDetailsByOrderRoute,
		getAll: getAllOrderDetailsRoute,
		update: updateOrderDetailRoute,
		delete: deleteOrderDetailRoute,
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

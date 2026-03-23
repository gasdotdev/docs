# D1 Seeding

Seeders exist in the corresponding resource's `_seed.ts` file and are ran:

- On `gas dev:start`, after the plugin's local dev D1 instance starts and migrations finish.
- On D1 migration file changes in dev mode, after migrations re-run and seeded data is reloaded.
- On `_seed.ts` create/update in dev mode.
- During deployment, against the remote D1 database, if seeding is configured in the resource's
`params.ts` file.

## Define Seeding

Seeding is configured in the D1 resource's `params.ts` file using the `seed` property.

`gas/db/params.ts`:
```ts
import { defineCfD1 } from '@gasdotdev/plugin-cloudflare/definitions';

export const rootDb = (() => {
	return defineCfD1({
		name: 'root-db',
		migrate:
			process.env.GAS_STAGE === 'production'
				? {
						onChange: './migrations',
					}
				: undefined,
		seed:
			process.env.GAS_STAGE === 'preview'
				? {
						onChange: ['./migrations', './src/**/*.ts'],
					}
				: undefined,
	} as const);
})();
```

## Example _seed.ts

`gas/db/src/_seed.ts`:
```ts
import { D1Seeder } from '@gasdotdev/plugin-cloudflare/d1-seeder';
import { rootDb } from '../params.ts';
import { faker } from '@faker-js/faker';
import { createAuthor } from './author.ts';
import { createBook } from './book.ts';

export async function seed() {
	// Keep data consistent across seeds
	faker.seed(123);

	const d1Seeder = new D1Seeder({
		resourceName: rootDb.name,
	});

	await d1Seeder.start();

	const db = d1Seeder.instance();

	console.log('running seeder');

	const authorIds: string[] = [];

	console.log('creating authors...');

	for (let i = 0; i < 10; i++) {
		const result = await createAuthor({
			db,
			author: {
				name: faker.person.fullName(),
			},
		});
		if (result.ok) authorIds.push(result.val.id);
	}

	console.log('creating books...');

	for (let i = 0; i < 50; i++) {
		await createBook({
			db,
			book: {
				authorId: faker.helpers.arrayElement(authorIds),
				title: faker.lorem.words({ min: 2, max: 5 }),
			},
		});
	}

	await d1Seeder.stop();

	console.log('ran seeder');
}

(async () => {
	try {
		await seed();
	} catch (error) {
		console.error('Unable to seed database:', error);
		process.exit(1);
	}
})();
```

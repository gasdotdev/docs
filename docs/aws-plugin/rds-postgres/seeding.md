# AWS RDS Postgres Seeding

Seeders exist in the corresponding resource's `_seed.ts` file and are ran:

- On `gas dev:start`, after the plugin's dev Postgres container starts and migrations finish.
- On Postgres migration file changes in dev mode, after the database is reset and migrations re-run.
- On `_seed.ts` create/update in dev mode.
- During deployment, against the remote RDS Postgres database, if seeding is configured in the resource's `params.ts` file.

## Example _seed.ts

`gas/api/src/_seed.ts`:
```ts
import pg from 'pg';
import { faker } from '@faker-js/faker';
import { createAuthor } from './authors.ts';
import { createBook } from './books.ts';

const connectionString = process.argv[2];

if (!connectionString) {
  throw new Error(
    'Unable to seed; missing connection string arg; node _seed.ts <connection-string>',
  );
}

async function seed() {
  faker.seed(123);

  const db = new pg.Client({ connectionString });
  await db.connect();

  console.log('running seeder');

  const authorIds: string[] = [];

  console.log('creating authors...');

  for (let i = 0; i < 5; i++) {
    const result = await createAuthor({
      db,
      author: {
        name: faker.person.fullName(),
      },
    });
    if (!result.err) authorIds.push(result.id);
  }

  console.log('creating books...');

  for (let i = 0; i < 15; i++) {
    await createBook({
      db,
      book: {
        authorId: faker.helpers.arrayElement(authorIds),
        title: faker.lorem.words({ min: 2, max: 5 }),
      },
    });
  }

  await db.end();

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

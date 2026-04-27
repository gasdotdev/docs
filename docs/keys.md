# Keys

The preferred pattern for encrypting values with Gas.

## Define Keys

Put keys in `gas/_keys/index.ts`.

`gas/_keys/index.ts`:
```ts
import { defineKeys } from '@gasdotdev/gas/config';

export const keys = defineKeys({
	development: {
		public: {
			from: 'file',
			path: './development.public',
		},
		private: {
			from: 'file',
			path: './development.private',
		},
	},
	production: {
		public: {
			from: 'file',
			path: './production.public',
		},
		private: {
			from: 'env',
			name: 'GAS_PRIVATE_KEY_PRODUCTION',
		},
	},
} as const);
```

## Use Keys in `gas.config.ts`

`gas.config.ts`:
```ts
import { defineConfig } from '@gasdotdev/gas/config';
import { cloudflare } from '@gasdotdev/plugin-cloudflare';
import { keys } from './gas/_keys';

export default defineConfig({
	containerDirPath: './gas',
	projectName: 'my-app',
	keys,
	plugins: [
		cloudflare({
			workersDevSubdomain: 'my-app',
			credentials: {
				accountId: `encrypted:${keys.names.production}:abcdefghijk`,
				apiToken: `encrypted:${keys.names.production}:lmnopqrstuv`,
			},
		}),
	],
});
```

## Use Keys in `params.ts`

`gas/api/params.ts`:
```ts
import { defineCfWorkerApi } from '@gasdotdev/plugin-cloudflare/definitions';
import { keys } from '../_keys';

export async function getRootApiParams() {
	return defineCfWorkerApi({
		env: {
			plaintext: {
				GREETING: 'hello',
			},
			secret: {
				SECRET_KEY: `encrypted:${keys.names.development}:abcdefghijk`,
			},
		},
		name: 'root-api',
		workersDev: true,
	} as const);
}
```

## Format

Encrypted values are plain strings:

```ts
encrypted:production:abcdefghijk
```

This means:

- the value is encrypted
- it was encrypted with the `production` public key
- Gas should use the `production` private key to decrypt it

## Notes

- Key names are not forced to match stages.
- Public keys are usually safe to commit.
- Private keys should stay local or come from env vars.
- `keys.names.production` is just the string `production`, but type-safe.

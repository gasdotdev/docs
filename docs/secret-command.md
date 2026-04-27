# Secret Command

Encrypt a value with a configured key.

## Basic Usage

```sh
gas secret
gas secret --key production
gas secret --key production --value "my-secret"
```

## Output

```txt
encrypted:production:abcdefghijk
```

## Example

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

Encrypt a value:

```sh
gas secret --key production --value "super-secret-value"
```

Then use it:

`gas/api/params.ts`:
```ts
import { defineCfWorkerApi } from '@gasdotdev/plugin-cloudflare/definitions';
import { keys } from '../_keys';

export async function getRootApiParams() {
	return defineCfWorkerApi({
		env: {
			secret: {
				SECRET_KEY: `encrypted:${keys.names.production}:abcdefghijk`,
			},
		},
		name: 'root-api',
		workersDev: true,
	} as const);
}
```

## Notes

- If `--key` is omitted, Gas will prompt for it.
- If `--value` is omitted, Gas will prompt for it.
- The command uses the selected key's public key.

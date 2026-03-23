# Gas.dev

## ?

A framework for creating, deploying, and managing Cloudflare-infused projects with type-safe code.

## Requirements

- dnsmasq for localhost domain routing (see [Installation](./docs/dnsmasq/installation.md) to set up)

## AWS Plugin

### RDS Postgres

- [Migrations](./docs/aws-plugin/rds-postgres/migrations.md)
- [Pattern - Entities](./docs/aws-plugin/rds-postgres/pattern-entities.md)
- [Seeding](./docs/aws-plugin/rds-postgres/seeding.md)

## Cloudflare Plugin

### D1

- [Migrations](./docs/cloudflare-plugin/d1/migrations.md)
- [Pattern - Entities](./docs/cloudflare-plugin/d1/pattern-entities.md)
- [Seeding](./docs/cloudflare-plugin/d1/seeding.md)

### Worker API

- [ORPC - Docs](./docs/cloudflare-plugin/worker-api/orpc-docs.md)
- [Pattern - D1](./docs/cloudflare-plugin/worker-api/pattern-d1.md)
- [Pattern - Hyperdrive + Postgres](./docs/cloudflare-plugin/worker-api/pattern-hyperdrive-postgres.md)
- [Testing - D1](./docs/cloudflare-plugin/worker-api/testing-d1.md)
- [Testing - Postgres](./docs/cloudflare-plugin/worker-api/testing-postgres.md)

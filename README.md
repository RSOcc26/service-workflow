# service-workflow

Workflow service: the catalog of articles and operations, projects, work orders with versioned operation graphs, and work-order release.

Part of **CompanyCentral**, a multi-tenant, cloud-native platform built for
the *Cloud Computing Services* course (RSO 2026/27, FRI UL).
See the [architecture](https://github.com/RSOcc26/.github/blob/main/docs/architecture.md) and the
[RSOcc26 organization](https://github.com/RSOcc26) for the other services.

> **Status:** scaffold — no implementation yet. API resources and events below
> are the planned design, not working features.

## Responsibilities

- Catalog of articles (products and materials) and operations
- Projects grouping work orders
- Work orders: article, quantity, due date, bill of materials
- Versioned operation graphs (routings): which operations, in which order
- Releasing a work order to production, with a snapshot of its graph version

## Data ownership

This service owns its own PostgreSQL database; no other service reads or
writes it directly ([ADR-0003](https://github.com/RSOcc26/.github/blob/main/docs/adr/0003-database-per-service.md)).

Owns: articles, operations, projects, work orders, operation graphs and their versions.

Multi-tenancy: tenants share tables. Every tenant-owned row has a `tenant_id`,
and PostgreSQL row-level security restricts each transaction to the tenant
from the verified tenant context ([ADR-0004](https://github.com/RSOcc26/.github/blob/main/docs/adr/0004-multi-tenancy-shared-tables-rls.md),
[ADR-0008](https://github.com/RSOcc26/.github/blob/main/docs/adr/0008-tenant-context-validation.md)). All data is tenant-scoped.

## API

The OpenAPI 3.1 contract is [`src/api/openapi.yaml`](src/api/openapi.yaml) and
will be served at `/api-docs`. It currently defines only the operational
endpoints; resources are added contract-first, together with their
implementation.

| Interface | Description |
|---|---|
| Health | `/health/live` and `/health/ready` for Kubernetes probes |
| Metrics | `/metrics` in Prometheus format |
| API docs | `/api-docs`, the OpenAPI document |

Planned resources (through the gateway as `/api/workflow/v1/...`):

| Resource | Path | Purpose |
|---|---|---|
| Articles | `/v1/articles` | Catalog of products and materials |
| Operations | `/v1/operations` | Catalog of operation types |
| Projects | `/v1/projects` | Projects grouping work orders |
| Work orders | `/v1/work-orders` | Create, plan and read work orders |
| Operation graphs | `/v1/work-orders/{id}/graph-versions` | Versioned operation graphs of a work order |
| Release | `/v1/work-orders/{id}/release` | Release a work order to production |

## Events

Published through the transactional outbox to the `companycentral.events`
exchange; consumers are idempotent
([ADR-0006](https://github.com/RSOcc26/.github/blob/main/docs/adr/0006-rabbitmq-transactional-outbox.md)). Planned:

| Direction | Event | Description |
|---|---|---|
| Publishes | `workflow.work-order.released` | A work order was released, with its graph snapshot and material needs |
| Publishes | `workflow.work-order.cancelled` | A released work order was cancelled |
| Consumes | `production.operation.completed` | Track progress of the work order |
| Consumes | `production.work-order.completed` | Mark the work order completed |

## Configuration

All configuration comes from environment variables; see
[`.env.example`](.env.example). Secrets are never committed; in Kubernetes
they come from Secrets, everything else from ConfigMaps.

| Variable | Required | Description |
|---|---|---|
| `PORT` | no | HTTP port (default `3000`) |
| `LOG_LEVEL` | no | `debug`, `info`, `warn` or `error` (default `info`) |
| `DATABASE_URL` | yes | PostgreSQL URL of this service's database, runtime role |
| `DATABASE_MIGRATION_URL` | yes | Same database, migration role (used only by migrations) |
| `RABBITMQ_URL` | yes | RabbitMQ connection URL |
| `RABBITMQ_EXCHANGE` | no | Event exchange (default `companycentral.events`) |
| `TENANT_CONTEXT_JWKS_URL` | yes | Where to get the gateway's public keys for verifying the tenant context |
| `TENANT_CONTEXT_ISSUER` | yes | Expected issuer of the tenant context |
| `TENANT_CONTEXT_AUDIENCE` | yes | Expected audience (`service-workflow`) |

## Getting started

Prerequisites: Node.js 24 (see `.nvmrc`), Docker, and the local stack from the
[deploy](https://github.com/RSOcc26/deploy) repository (PostgreSQL, RabbitMQ).

_Commands will be added with the first implementation._

## Project structure

```
src/
  api/          HTTP routes, request validation, OpenAPI document
  domain/       business rules, independent of HTTP and storage
  data/         database access (repositories, queries)
  events/       outbox publisher and event consumers
  config/       configuration loading and validation
migrations/     versioned database migrations
test/
  unit/         fast tests with no external dependencies
  integration/  tests against a real database
docs/
  adr/          architecture decision records
```

## Deployment

Built as a container image and published to GitHub Container Registry.
Kubernetes manifests and Helm charts live in the deployment repository.

## Contributing

- Branch from `main` (`feat/…`, `fix/…`, `chore/…`) and open a pull request.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/).
- Pull requests need a review from a code owner and passing checks; they are squash-merged.
- Record notable changes in `CHANGELOG.md`.

## License

[MIT](LICENSE)

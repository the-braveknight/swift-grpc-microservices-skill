# Architecture specification

Use a service-oriented SwiftPM package, not textbook layer names. The package is the deployment unit; targets are compile-time boundaries inside it.

## Target graph

```text
<Service>
├── <Service>Core
├── <Service>Postgres ─────→ <Service>Core
├── <Service>GRPC ─────────→ <Service>Core
├── <Service>Workflows ────→ <Service>Core  # only with Temporal
└── <Service> ─────────────→ all required targets

```

Use these exact responsibilities:

| Target | Owns | Must not own |
| --- | --- | --- |
| `<Service>Core` | Entities, repository protocols and errors, repository commands, database protocol, use-case protocols/inputs/errors/contexts/implementations | SQL, generated messages, RPC status, configuration, logging, server startup |
| `<Service>Postgres` | Postgres context/database, repositories, prepared statements, migrations | Business validation, RPC mapping, CLI parsing |
| `<Service>GRPC` | Generated-service conformances, request/input mapping, domain/protobuf mapping, RPC error translation | SQL, environment reading, server construction |
| `<Service>Workflows` | Temporal Workflows, Activity containers, workflow-client adapters, signals, queries, and Temporal error translation | SQL implementations, generated protobuf, environment reading, dependency construction |
| `<Service>` | ArgumentParser command tree, environment configuration, logging, dependency construction, server/client construction, lifecycle | Reusable business rules |

Keep dependencies pointing inward. Core has no dependency on another service's implementation. If Core genuinely needs an external type as part of an application port, depend only on the smallest contract library and document why; do not import an entire infrastructure SDK for convenience.

## Naming grammar

- Service package: `<organization>-<service>` or the repository's established lowercase service naming pattern.
- Executable product: lowercase service name, such as `catalog`.
- Internal targets: `CatalogCore`, `CatalogPostgres`, `CatalogGRPC`, `Catalog`.
- Optional Temporal target: `CatalogWorkflows`.
- Entity: `Item`.
- Use case: `CreateItemUseCase`.
- Use-case port: `CreateItemUseCaseProtocol`.
- Input and typed failure: `CreateItemUseCaseInput`, `CreateItemUseCaseError`.
- Narrow context: `CreateItemUseCaseContext`.
- Repository: `ItemRepository`, `ItemRepositoryError`.
- Write intent passed to a repository: `CreateItemCommand`.
- Postgres implementation: `PostgresItemRepository`.
- Prepared statement: `CreateItemStatement`.
- gRPC service implementation: `ItemService`.
- Temporal workflow, Activities, and client adapter: `ReservationWorkflow`, `ReservationActivities`, `TemporalReservationWorkflowClient`.

Do not replace these with handlers, interactors, gateways, stores, managers, `Domain`, `Application`, `Infrastructure`, generic `Mappings`, or generic `Adapters` unless the user explicitly changes the vocabulary.

## Boundary rules

Use protocols where they enable a real seam:

- Use-case protocols let transports and callers depend on behavior.
- Repository protocols let Core depend on persistence capabilities.
- Per-use-case context protocols expose only the repositories a use case needs.
- `Database` lets the use case select connection versus transaction semantics while keeping Core decoupled from persistence.
- Workflow-client protocols let Core start, signal, and query durable orchestration without importing Temporal.
- Activity service protocols let the Workflows target invoke Core-owned application behavior without importing Postgres, gRPC, email, or provider SDKs.

Do not add a protocol merely to mirror every concrete type. Keep entities and command values as structs. Keep configuration and composition concrete at the executable root.

Use typed errors in Core. Map persistence-specific failures into repository errors in persistence, repository errors into use-case errors in Core, and use-case errors into RPC status in gRPC. Reverse that translation at the consumer boundary. Never make Core switch on SQLSTATE or `RPCError`.

## Ownership rules

An extracted service owns:

- its business behavior;
- its canonical RPC surface;
- its database and migrations;
- its persistence timestamps and constraints;
- its runtime and deployment lifecycle.

The database that owns an entity also owns generation of that entity's identifier. Define UUID primary keys with `DEFAULT uuidv7()`, omit them from create commands and create RPC requests, and return the inserted entity with its generated identifier. A caller may provide a separate idempotency key when required, but it must not masquerade as the owned entity identifier. Treat the key as an opaque request identity, not a secret or authorization credential; the receiving service owns atomic enforcement and payload-conflict detection.

When one service creates an entity through another service, keep the caller's pending state under a caller-owned identifier. Store the foreign identifier only after the owning service returns it. Never preallocate an identifier in the caller, pass it into the owner, or create a matching identifier independently in two databases.

The monolith or another consumer owns its local API-facing model and use-case protocol. Generated protobuf messages are integration DTOs, not shared domain entities.

Do not share a database, query another service's tables, or make a distributed transaction. Add an RPC for synchronous coordination. Introduce asynchronous events/outbox only when delivery and consistency requirements justify it; do not add them speculatively.

When Temporal coordinates a capability, keep orchestration mechanics in `<Service>Workflows` and business/persistence state in Core and Postgres. Do not mirror Temporal history in a second workflow-state infrastructure or poll Postgres to reconstruct running workflows by default. See [temporal-workflows.md](temporal-workflows.md).

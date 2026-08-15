# Architecture specification

Use a service-oriented SwiftPM package, not textbook layer names. The package is the deployment unit; targets are compile-time boundaries inside it.

## Target graph

```text
<Service>
├── <Service>Core
├── <Service>Postgres ─────→ <Service>Core
└── <Service>GRPC ─────────→ <Service>Core

<Service>CoreTests ────────→ <Service>Core
```

Use these exact responsibilities:

| Target | Owns | Must not own |
| --- | --- | --- |
| `<Service>Core` | Entities, repository protocols and errors, repository commands, database protocol, use-case protocols/inputs/errors/contexts/implementations | SQL, generated messages, RPC status, configuration, logging, server startup |
| `<Service>Postgres` | Postgres context/database, repositories, prepared statements, migrations | Business validation, RPC mapping, CLI parsing |
| `<Service>GRPC` | Generated-service conformances, request/input mapping, domain/protobuf mapping, RPC error translation | SQL, environment reading, server construction |
| `<Service>` | ArgumentParser command tree, environment configuration, logging, dependency construction, server/client construction, lifecycle | Reusable business rules |
| `<Service>CoreTests` | Use-case tests and local mocks | Imports of Postgres, grpc-swift, protobuf, configuration, logging, or the executable |

Keep dependencies pointing inward. Core has no dependency on another service's implementation. If Core genuinely needs an external type as part of an application port, depend only on the smallest contract library and document why; do not import an entire infrastructure SDK for convenience.

## Naming grammar

- Service package: `<organization>-<service>` or the repository's established lowercase service naming pattern.
- Executable product: lowercase service name, such as `catalog`.
- Internal targets: `CatalogCore`, `CatalogPostgres`, `CatalogGRPC`, `Catalog`.
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

Do not replace these with handlers, interactors, gateways, stores, managers, `Domain`, `Application`, `Infrastructure`, generic `Mappings`, or generic `Adapters` unless the user explicitly changes the vocabulary.

## Boundary rules

Use protocols where they enable a real seam:

- Use-case protocols let transports and callers depend on behavior.
- Repository protocols let Core depend on persistence capabilities.
- Per-use-case context protocols expose only the repositories a use case needs.
- `Database` lets the use case select connection versus transaction semantics and makes Core tests fast.

Do not add a protocol merely to mirror every concrete type. Keep entities and command values as structs. Keep configuration and composition concrete at the executable root.

Use typed errors in Core. Map persistence-specific failures into repository errors in persistence, repository errors into use-case errors in Core, and use-case errors into RPC status in gRPC. Reverse that translation at the consumer boundary. Never make Core switch on SQLSTATE or `RPCError`.

## Ownership rules

An extracted service owns:

- its business behavior;
- its canonical RPC surface;
- its database and migrations;
- its persistence timestamps and constraints;
- its runtime and deployment lifecycle.

The monolith or another consumer owns its local API-facing model and use-case protocol. Generated protobuf messages are integration DTOs, not shared domain entities.

Do not share a database, query another service's tables, or make a distributed transaction. Add an RPC for synchronous coordination. Introduce asynchronous events/outbox only when delivery and consistency requirements justify it; do not add them speculatively.

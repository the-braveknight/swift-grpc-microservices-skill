# Architecture specification

Use a service-oriented SwiftPM package, not textbook layer names. The package is the deployment unit; targets are compile-time boundaries inside it.

## Target graph

```text
<Service>
├── <Service>Core
├── <Service>Postgres ─────→ <Service>Core
├── <Service>GRPC ─────────→ <Service>Core
├── <Service>Workflows ────→ <Service>Core  # only with Temporal
├── <Service>Worker ───────→ Core, GRPC, Workflows, providers  # only with Temporal; a second executable
├── <Service><Technology> ─→ <Service>Core  # one per third-party provider SDK
└── <Service> ─────────────→ all required targets
```

| Target | Owns | Must not own |
| --- | --- | --- |
| `<Service>Core` | Entities, repository protocols and errors, repository commands, database protocol, policies and validators, use-case protocols/inputs/errors/contexts/implementations, workflow-client and Activity-service ports | SQL, generated messages, RPC status, configuration, server startup, a logging backend |
| `<Service><Technology>` | One provider SDK's adapter conforming to a Core port, such as `<Service>Bcrypt` or `<Service>Resend` | Business rules, configuration reading, client construction |
| `<Service>Postgres` | Postgres context/database, repositories, prepared statements, migrations | Business validation, RPC mapping, CLI parsing |
| `<Service>GRPC` | Generated-service conformances, request/input mapping, domain/protobuf mapping, RPC error translation, per-RPC identity checks | SQL, environment reading, server construction |
| `<Service>Workflows` | Temporal Workflows, Activity containers, workflow-client adapters, signals, queries, Temporal error translation | SQL implementations, generated protobuf, environment reading, dependency construction |
| `<Service>` | ArgumentParser command tree, environment configuration, logging bootstrap, dependency construction, server/client construction, lifecycle | Reusable business rules |
| `<Service>Worker` | The Temporal worker's command, configuration, logging bootstrap, client and worker construction, lifecycle — a second composition root | A database client, a server, business rules |

## Dependency direction

Dependencies point inward. Core has no dependency on another service's implementation. Zero dependencies is not the rule — Swift server libraries routinely ship a small-dependency core — and Core may link a focused library such as `swift-crypto`, the `swift-log` facade, or the organization's shared identity contract. What Core must not link is an infrastructure SDK: a database driver, an HTTP client, a mail or payments provider, a server framework, a concrete logging backend.

A third-party SDK is what earns a `<Service><Technology>` target; conforming to a Core protocol does not. Keep those targets leaves that only the composition root depends on. Core is the root of the internal graph, so a mail SDK there makes `<Service>Postgres` link an HTTP client to run SQL, and an email-template edit recompiles every target.

## Business rules, policies, and adapters

A business rule lives in the use case that applies it. A fixed invariant — a title must not be blank, a currency is three letters, a licence names a major version and a subscription does not — is a `guard` at the top of `callAsFunction`, before any I/O, throwing the use case's own typed error. Do not extract those guards into an `XValidator` whose errors then need an initializer on every use-case error to map them back; the mapping is code that says nothing. When the same invariant applies on create and on update, both use cases state it, and a rule that depends on the row's existing state is applied inside the transaction that reads the row.

An adapter carries no rule. A `<Service><Technology>` target translates one Core call into one SDK call and the SDK's result and errors back into Core values; it does not decide whether a trial is offered, whether a message should be sent, or whether a caller qualifies. The use case makes that decision from the entities it already holds and hands the adapter a plain value — a `trialPeriodDays: Int?` that is `nil` when there is no trial, not a `trialEligible` flag and a `requestTrial` flag for the adapter to combine. The same holds for a repository: SQL states constraints and guards transitions in `WHERE`, but a decision that reads as product policy belongs above it.

A rule that carries product-set values — a minimum length, a lifetime — is a plain struct in Core, never a protocol, never injected:

```swift
package struct PasswordPolicy: Equatable, Sendable {
    package static let standard = PasswordPolicy()
    package let minimumLength: Int
    package let requiresNumber: Bool
}
```

Name a rule expressed as numbers `XPolicy`, not `XConfiguration`. Configuration is deployment wiring that varies per environment; a policy is a product decision that would be identical in staging and production. The composition root reads configuration and translates it into policy — being loadable from `ConfigReader` does not make a value configuration. Name a duration `expiration`, matching the `expirationDate` it produces.

Do not inject a concrete collaborator that has no protocol: injection buys substitution, and a concrete type cannot be substituted. Either the seam is real and the parameter is a protocol, or the type is constructed where it is used. A policy is the exception because it is data, not a collaborator: the composition root builds it from configuration and passes it by value into the use case's initializer, which defaults the parameter to `.standard` so a test constructs the use case without it. A validator that applies such a policy — `PasswordValidator(policy:)` — travels the same way; a validator with no values to carry is not a type at all but a guard in the use case.

## Duplicate the small shapes

Duplicate the `Database`, `PostgresContext`, `PostgresDatabase`, configuration, transport-security, and composition skeletons in each service rather than extracting a shared foundation package. The shapes start identical but are service-owned: contexts, database wrappers, and configuration diverge as services evolve, and a shared package couples independent releases — every change costs a tag and a bump in every consumer. The only shared packages are the proto contracts and the identity package, and both are consumed by tag. Do not introduce an `<organization>-kit` unless the user explicitly asks for one.

## Naming grammar

- Service package: `<organization>-<service>`, lowercase.
- Executable product: the lowercase service name, such as `catalog`.
- Internal targets: `CatalogCore`, `CatalogPostgres`, `CatalogGRPC`, `Catalog`. Targets do not repeat the organization prefix; the package name carries it.
- Optional Temporal target: `CatalogWorkflows`.
- Provider adapter target: `CatalogBcrypt`, `CatalogResend` — the technology, not the port it implements.
- Business rule value: `PasswordPolicy`, `SessionPolicy`, `EmailValidator`.
- Entity: `Item`.
- Use case: `CreateItemUseCase`; port `CreateItemUseCaseProtocol`; input and typed failure `CreateItemUseCaseInput`, `CreateItemUseCaseError`; narrow context `CreateItemUseCaseContext`.
- Repository: `ItemRepository`, `ItemRepositoryError`; write intent `CreateItemCommand`.
- Postgres implementation: `PostgresItemRepository`; prepared statement `CreateItemStatement`.
- gRPC service implementation: `ItemService`.
- Temporal workflow, Activities, and client adapter: `ReservationWorkflow`, `ReservationActivities`, `TemporalReservationWorkflowClient`.
- Identifier properties: `xId`, never `xID`. Strict camel case keeps `registrationId` aligned with SQL `registration_id` and proto `registration_id` with no acronym special-casing.

Do not replace these with handlers, interactors, gateways, stores, managers, `Domain`, `Application`, `Infrastructure`, generic `Mappings`, or generic `Adapters` unless the user explicitly changes the vocabulary.

## Boundary rules

Use protocols where they enable a real seam:

- Use-case protocols let transports and callers depend on behavior.
- Repository protocols let Core depend on persistence capabilities.
- Per-use-case context protocols expose only the repositories a use case needs.
- `Database` lets the use case select connection versus transaction semantics while keeping Core decoupled from persistence.
- Workflow-client protocols let Core start, signal, and query durable orchestration without importing Temporal.
- Activity service protocols let the Workflows target invoke Core-owned behavior without importing Postgres, gRPC, or provider SDKs.

Do not add a protocol merely to mirror every concrete type. Keep entities and command values as structs. Keep configuration and composition concrete at the executable root.

Use typed errors in Core. Map persistence failures into repository errors in persistence, repository errors into use-case errors in Core, and use-case errors into RPC status in gRPC. Reverse that translation at the consumer boundary. Never make Core switch on SQLSTATE or `RPCError`.

Translate an error only where the translation carries information. A one-case adapter enum every failure funnels into — `catch { throw XClientError.unavailable }` — destroys the cause without adding a distinction, so a permanent misconfiguration reports as a retryable outage and the caller retries what can never succeed. Let the infrastructure error propagate and classify it where the difference is actionable. Name a repository error for the constraint that exists: a partial unique index over active states means "an active record exists", not "duplicate email".

## Ownership rules

A service owns:

- its business behavior;
- its canonical RPC surface;
- its database and migrations;
- its persistence timestamps, identifiers, and constraints;
- its runtime and deployment lifecycle.

The database that owns an entity also owns generation of that entity's identifier. Define UUID primary keys with `DEFAULT uuidv7()`, omit them from create commands and create RPC requests, and return the inserted entity with its generated identifier. A caller may provide a separate idempotency key when required, but it must not masquerade as the owned entity identifier. Treat the key as an opaque request identity, not a secret or authorization credential; the receiving service owns atomic enforcement and payload-conflict detection.

When one service creates an entity through another service, keep the caller's pending state under a caller-owned identifier. Store the foreign identifier only after the owning service returns it. Never preallocate an identifier in the caller, pass it into the owner, or create a matching identifier independently in two databases.

A consumer owns its local caller-facing model and use-case protocol. Generated protobuf messages are integration DTOs, not shared domain entities.

Do not share a database, query another service's tables, or make a distributed transaction. Add an RPC for synchronous coordination. Introduce asynchronous events or an outbox only when delivery and consistency requirements justify it; do not add them speculatively.

When Temporal coordinates a capability, keep orchestration mechanics in `<Service>Workflows` and business/persistence state in Core and Postgres. Do not mirror Temporal history in a second workflow-state store or poll Postgres to reconstruct running workflows. See [temporal-workflows.md](temporal-workflows.md).

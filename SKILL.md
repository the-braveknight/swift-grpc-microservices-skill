---
name: swift-grpc-microservices-skill
description: Design, build, deploy, and evolve production Swift gRPC microservices and distributed systems using strict reusable module and code conventions. Use when creating a service or multi-service system from scratch, defining service boundaries and communication patterns, structuring SwiftPM packages, designing versioned protobuf contracts, implementing Postgres ownership and transactions, wiring grpc-swift clients or servers, preserving XUseCase/XRepository/XCommand boundaries, adding composition roots and Docker Compose deployment, or extracting bounded contexts from a modular monolith.
---

# Swift gRPC Microservices

Use this skill as an end-to-end architecture and implementation specification for greenfield Swift distributed systems, individual microservices, and modular-monolith extractions. Preserve its naming, module boundaries, folder structure, composition roots, formatting, and deployment topology unless the user explicitly changes a convention.

## Load the required references

Read these files before changing code:

- Read [architecture.md](references/architecture.md) for every task.
- Read [swift-style.md](references/swift-style.md) before creating or editing Swift.
- Read [service-package.md](references/service-package.md) when creating or reshaping a service package.
- Read [core.md](references/core.md) when implementing entities, repositories, commands, use cases, or contexts.
- Read [persistence.md](references/persistence.md) when implementing Postgres, transactions, statements, repositories, or migrations.
- Read [grpc-and-protos.md](references/grpc-and-protos.md) when defining contracts or implementing producer and consumer transports.
- Read [composition-and-deployment.md](references/composition-and-deployment.md) when wiring executables, configuration, lifecycle, containers, or Compose.
- Read [distributed-systems.md](references/distributed-systems.md) for every greenfield multi-service system and every task involving cross-service communication, consistency, resilience, security, or observability.
- Read [extraction-runbook.md](references/extraction-runbook.md) in full when extracting a bounded context or decomposing an existing monolith.

## Follow the governing rules

1. Name service targets `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`; never substitute `Domain`, `Application`, or `Infrastructure`.
2. Use `XUseCase`, `XUseCaseProtocol`, `XUseCaseInput`, `XUseCaseError`, `XUseCaseContext`, `XRepository`, `XRepositoryError`, and repository-facing `XCommand` names.
3. Keep Core independently buildable. It must not import Postgres, gRPC, protobuf, logging, configuration, or server libraries.
4. Keep the `Database` protocol in Core. Use `withConnection` for a single repository operation and `withTransaction` only for a multi-operation atomic use case. Never hold a database transaction across an RPC.
5. Let repositories or the database stamp persistence-owned dates. Use a database default such as `DEFAULT NOW()` in Postgres. Do not inject a `now` closure into a use case.
6. Start with primitive domain values when sufficient. Introduce value objects only for established invariants or behavior, not architectural ceremony.
7. Give every service sole ownership of its database. Use unqualified table names and no service-named Postgres schema. Name migrations for the database change without a redundant service prefix.
8. Put protobuf conversions in the transport feature's `Protobuf/` directory using semantic `X+Protobuf.swift` names. Keep the generated-service conformance at the feature root. Do not create generic `Mappings` or `Adapters` directories.
9. Store canonical `.proto` files only in the shared `<project>-protos` Swift package at `https://github.com/<organization>/<project>-protos.git`. Nest proto files by organization, service, and version. Do not copy schemas into producers or consumers.
10. Use `package` access between targets in one service package. Expose `public` API only from separately consumed packages or established cross-target APIs.
11. Keep caller-facing use-case protocols and local entities at consumer boundaries. Do not leak generated messages into HTTP handlers or Core business code.
12. Give every target its direct dependencies in `Package.swift`; never rely on transitive imports.
13. Use gRPC for synchronous capability calls. Add asynchronous messaging, workflow orchestration, outbox delivery, or projections only when product consistency and delivery requirements justify them. Never share databases or create distributed transactions.
14. Construct long-lived clients and servers once in executable composition roots and manage them with `ServiceGroup`, graceful shutdown, scoped configuration, and structured logging.
15. Keep internal container traffic plaintext when a trusted private network and ingress terminate TLS. Add service-level TLS or mTLS when the threat model requires it; do not expose internal gRPC ports publicly by default.
16. For existing systems, preserve behavior and unrelated user changes. For greenfield systems, state consequential assumptions and choose the smallest architecture that satisfies current requirements.

## Choose the workflow

- For one new service, follow **Build a service**.
- For a new distributed system, follow **Design and build a system**, then repeat **Build a service** for each bounded context.
- For an existing modular monolith, follow **Extract an existing bounded context** and the extraction runbook.
- For a focused change, load only the relevant references and preserve the established architecture.

## Design and build a system

1. Capture business capabilities, external actors, scale, latency, availability, security, compliance, and consistency requirements.
2. Define bounded contexts and data ownership before repositories or RPCs. Produce a service map showing callers, dependencies, and failure boundaries.
3. Choose local calls, synchronous gRPC, asynchronous events, or durable workflow coordination per interaction. Document why each remote boundary exists.
4. Define versioned contracts and failure semantics before implementing consumers and producers.
5. Create the shared proto package, then initialize each independently deployable service with the required SwiftPM target structure.
6. Build vertical capabilities through Core, Postgres, gRPC, executable composition, and deployment rather than creating empty horizontal layers across every service.
7. Add deadlines, idempotency, retry rules, observability, health behavior, security, deployment ordering, and failure handling in proportion to the system requirements.

## Build a service

1. Always run `swift package init --type executable` before adding targets, folders, or code.
2. Reshape the package to `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`.
3. Implement feature-first Core entities, commands, repositories, context protocols, use-case contracts, typed errors, and use cases.
4. Implement Postgres database/context, prepared statements, repositories, constraints, error translation, and ordered migrations.
5. Implement generated gRPC service protocols and feature-local `Protobuf/` conversion directories.
6. Add the `serve` and `database migrate` command composition roots, configuration, logging, and lifecycle management.
7. Add `.env.example`, container build support, and Compose services for Postgres, migration, and the service.
8. Build the complete package, then exercise contracts and infrastructure boundaries.

## Extract an existing bounded context

1. Inspect repository instructions, dirty state, package graph, callers, persistence, migrations, jobs, configuration, and deployment.
2. Trace the bounded context end to end and identify cross-context reads, transactions, shared types, and compatibility seams.
3. Define and release the versioned contract, build the producer as a standalone service, then preserve the consumer-facing use-case protocol while replacing its implementation with a gRPC client.
4. Construct one long-lived client in the consumer composition root and run it in `ServiceGroup`.
5. Deploy a dedicated database, migration job, producer, and consumer in dependency order; move and reconcile data explicitly.
6. Remove old persistence only after traffic cutover and rollback criteria pass. Never infer permission to discard data.

## Enforce completion gates

Do not call a service or system complete until all applicable gates pass:

- Every service package builds under Swift 6 language mode.
- Generated protobuf types appear only in proto, transport, and client-boundary code.
- Postgres types appear only in Postgres and executable composition code.
- Single-operation use cases use `withConnection`; justified multi-operation units use `withTransaction`.
- Producers map typed failures to stable gRPC status codes and consumers map them to local contracts.
- Long-lived clients and servers are lifecycle-managed and shut down gracefully.
- Every service owns a dedicated database with applied migrations and no service-specific schema.
- Remote calls have explicit failure semantics; retries are limited to safe/idempotent operations.
- Contracts are versioned and compatibility-safe; canonical protos are not duplicated.
- Logging, metrics/tracing strategy, health behavior, secrets, and network exposure match the operating environment.
- Deployment ordering and representative dependency failures are verified.
- Extraction work leaves no retired direct database access after cutover.

If a gate requires an unresolved product, consistency, security, or data-migration decision, stop at the safe boundary and request that decision rather than inventing behavior.

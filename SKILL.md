---
name: swift-grpc-microservices-skill
description: Design, build, deploy, and evolve production Swift gRPC microservices and distributed systems with an opinionated, experience-derived architecture. Use when creating a service or multi-service system, defining service boundaries and communication, structuring SwiftPM service packages, designing versioned protobuf contracts, implementing Postgres ownership, row-level security and idempotent writes, wiring grpc-swift servers and clients with mutual TLS, issuing and verifying caller and service identities, fronting a system with a Hummingbird HTTP gateway, coordinating durable Temporal workflows and workers, composing executables and Docker Compose environments, or extracting bounded contexts from a modular monolith.
---

# Swift gRPC Microservices

An end-to-end architecture and implementation specification for Swift distributed systems: one service, a whole system, the HTTP surface in front of it, and the extraction of services from a monolith. It encodes a specific way of building — names, module boundaries, folder layout, composition roots, security posture, and deployment shape — learned from running such systems. Preserve every convention here unless the user explicitly changes it; where the user's repository already has an established convention that differs, the repository wins for unrelated code.

## Load the references

Read the reference that owns a topic before touching that topic. Each topic has exactly one home; other files point to it.

| Task | Read |
| --- | --- |
| Every task | [architecture.md](references/architecture.md) |
| Creating or editing Swift | [swift-style.md](references/swift-style.md) |
| Creating or reshaping a service package, manifest, or dependency set | [service-package.md](references/service-package.md) |
| Entities, commands, repositories, contexts, use cases, policies | [core.md](references/core.md) |
| Postgres, statements, transactions, migrations, the service role, row-level security, idempotent writes | [persistence.md](references/persistence.md) |
| Proto contracts, producer and consumer adapters, error-to-status mapping | [grpc-and-protos.md](references/grpc-and-protos.md) |
| Executables, configuration, logging bootstrap, lifecycle, transport-security factories | [composition.md](references/composition.md) |
| Signing keys, tokens, interceptors, middleware, caller propagation, service identities | [identity-and-access.md](references/identity-and-access.md) |
| The HTTP gateway: routing tiers, request contexts, OpenAPI, HTTP-to-RPC translation | [api-gateway.md](references/api-gateway.md) |
| Temporal workflows, Activities, clients, signals, queries, timers, workers | [temporal-workflows.md](references/temporal-workflows.md) — read in full |
| Service boundaries, communication choice, consistency, resilience, observability | [distributed-systems.md](references/distributed-systems.md) |
| Images, Compose, ports, secrets, certificates, the gateway's address, first start | [environment.md](references/environment.md) — the only file that describes the running environment |
| Extracting a bounded context from an existing monolith | [extraction-runbook.md](references/extraction-runbook.md) — read in full |

## Principles

The rules below follow from these. When a situation is not covered, decide from the principle.

1. **A service owns a capability and everything about its data.** Its database, identifiers, timestamps, constraints, migrations, contract, and lifecycle. Nothing else reads its tables; nothing else generates its identifiers.
2. **Dependencies point inward, and a third-party SDK earns its own target.** Core links focused libraries, never infrastructure SDKs. Conforming to a Core protocol does not earn a target; linking a vendor SDK does.
3. **A protocol exists only where substitution is real.** Use cases, repositories, per-use-case contexts, the database, workflow clients, Activity services. Entities, commands, policies, and configuration stay concrete.
4. **The database is the authority.** It generates identifiers and dates, enforces uniqueness, guards state transitions in the `WHERE` clause, and confines callers with policies. Application code never re-implements what a constraint states.
5. **Retry safety lives with the owner of the side effect.** An idempotency key is enforced atomically where the write happens — never in memory, never in a caller, never in workflow history.
6. **A token says who is calling; the service decides what they may do.** Roles travel; permissions do not. Identification is shared infrastructure; enforcement is per-RPC, per-service code.
7. **Key material is a file; configuration carries a path.** Private keys, certificates, and credentials are mounted, opened in the composition root, and fail loudly at startup naming the path.
8. **Nothing is plaintext and nothing is published.** Every internal connection is mutually authenticated by a CA the stack issues itself; no internal port reaches a host interface.
9. **Duplicate small shapes; share only contracts and identity, by tag.** A `Database` wrapper or a configuration extension is eight lines a service owns. A shared package costs a tag and a bump in every consumer.
10. **Choose the smallest architecture that satisfies current requirements.** gRPC is the default. Messaging, workflow orchestration, outboxes, and projections are added from a stated requirement, never speculatively.
11. **Translate an error only where the translation adds information.** A one-case enum every failure funnels into destroys the cause; let it propagate and classify where the distinction is actionable.
12. **Fail at startup, not at the first request.** Required configuration, key files, certificates, and credential exchanges are checked before the process serves.

## Governing rules

### Package, targets, and naming

1. Name targets `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`; add `<Service>Workflows` and the `<Service>Worker` executable only with Temporal, and one `<Service><Technology>` leaf per third-party provider SDK. Never `Domain`, `Application`, `Infrastructure`, `Mappings`, or `Adapters`.
2. Use `XUseCase`, `XUseCaseProtocol`, `XUseCaseInput`, `XUseCaseError`, `XUseCaseContext`, `XRepository`, `XRepositoryError`, `XCommand`, `XPolicy`, `XValidator`, `XStatement`, `PostgresXRepository`, `XService`, `XWorkflow`, `XActivities`, `TemporalXWorkflowClient`.
3. Keep Core independently buildable. It may link `swift-crypto`, the `swift-log` facade, or the shared identity package; never Postgres, gRPC, protobuf, a logging backend, configuration, a provider SDK, or a server framework.
4. Give every target its direct dependencies in `Package.swift`; never rely on transitive imports. Use `package` access between targets of one package; `public` only from separately consumed packages.
5. Depend on organization packages by tagged URL, never `.package(path:)`. Publish and tag the shared package before a service pins it.
6. Do not extract a shared foundation package for `Database`, contexts, configuration, or logging bootstrap. Duplicate the shape per service.

### Core

7. Model entities and commands as immutable `Sendable` structs. Start with primitive values; introduce a value object only for an established invariant or behavior.
8. Keep business rules as plain structs named `XPolicy` or `XValidator` — never protocols, never injected. Configuration is deployment wiring; a policy is a product decision the composition root translates configuration into. Name a duration `expiration`.
9. Do not inject a concrete type that has no protocol. Either the seam is real and the parameter is a protocol, or the type is constructed where it is used.
10. Keep the `Database` protocol in Core. Use `withConnection` for one repository operation and `withTransaction` only for a multi-operation atomic use case — except in a service that confines callers with row-level security, whose protocol declares `withTransaction` alone.
11. Never hold a database transaction across a remote call. The sole exception is rotating a single-use secret, with an explicit deadline on the call well under the pool's wait time.
12. Let the database stamp persistence-owned dates with `DEFAULT NOW()`. Do not inject a `now` closure or clock into a use case.
13. Inject a `Logger` into a use case that performs a meaningful mutation, and log the domain event there: `info` on success, `warning` on a known refusal, `error` on a genuine failure, with searchable identifiers as metadata and never a secret or payload.

### Persistence

14. Every service owns its whole database: unqualified table names, no service-named schema, migrations named for the change without a service prefix, one create migration per table, registered explicitly in dependency order.
15. Generate every entity identifier in the owning database with `UUID DEFAULT uuidv7()`. Omit it from create commands and create RPC requests; return it from the owner after insertion. Never preallocate or generate another service's identifier.
16. Name stored dates as nouns — `creation_date`, `update_date`, `expiration_date` — and the Swift forms `creationDate`, `updateDate`, `expirationDate`. Never `created_at` or `somethingAt`. Use `Id`, not `ID`, in Swift names.
17. Model a non-identifier secret — a verification or refresh token — as one Core value type that mints and digests. Persist only the SHA-256 digest, hand the value to its bearer once, give the column no generation default, delete the row at terminal state. Reserve bcrypt for user-chosen passwords and service credentials.
18. Make every remotely retryable mutation idempotent in the service that owns the side effect: a stable, namespaced key, enforced atomically by the database or the provider, returning the prior outcome for the same canonical input and rejecting reuse with different input. Never use the key as authorization or as an entity identifier.
19. Guard every state transition in the `WHERE` clause and treat the absent row as the lost race. Carry only the destination state in the command and derive its one legal predecessor from the state enum.
20. Enforce liveness-scoped uniqueness with a partial unique index over the active states, and name a repository error for the constraint that exists.
21. Make `CreateServiceRole` the first migration of every service. Migrations run as the owner; every other command connects as the service role.
22. Confine caller-owned rows with row-level security: policies in migrations, never a scope restated in statements; stamp each transaction with the caller's role and, for a person, user id, using bound `set_config(…, true)`; keep `NULLIF` in every policy; admit `service` beside `admin`; verify as the service role. The handler still answers what a policy cannot — `permissionDenied` for a named user who is not the caller.

### Contracts and transport

23. Keep canonical `.proto` files only in the shared `<project>-protos` package, nested by organization, service, and version, starting at `v1`. Never copy schemas into producers or consumers.
24. Put protobuf conversions in the transport feature's `Protobuf/` directory as `X+Protobuf.swift`; keep the generated-service conformance at the feature root. Perform transport validation — UUID parsing, enum recognition, range checks — in the conversion initializer.
25. Keep generated messages out of Core and out of HTTP handlers. A consumer keeps its caller-facing use-case protocol and local entities and replaces only the implementation.
26. Map typed use-case failures to stable gRPC status codes in the producer; map status codes back to the consumer's local errors. Never let `PSQLError`, SQLSTATE, or `RPCError` reach Core.
27. Format UUIDs lowercased on the wire. Refuse an unrecognized enum value rather than reading it as the least privilege.
28. Use gRPC for synchronous capability calls. Add messaging, workflow orchestration, outbox delivery, or projections only from a stated consistency or delivery requirement. Never share databases or create distributed transactions.

### Composition and configuration

29. Name the executable after the service. `serve` is the default subcommand; `database migrate` is nested; operator commands are subcommands. A Temporal worker is not a subcommand: it is a second executable target `<Service>Worker`, product `<service>-worker`, image `<organization>-<service>-worker`, in the same package, with its own composition root and its own copies of the configuration extensions it needs.
30. Construct every long-lived client, server, and worker once in the composition root, under `// MARK:` sections in the order Configuration, Logging, Infrastructure, Composition, Transport, Lifecycle, and own all of them with one `ServiceGroup` with graceful shutdown.
31. Read configuration with `ConfigReader` scoped by concern (`postgres`, `grpc.server`, `grpc.<upstream>`, `tls`, `jwt`, `temporal`, `loki`). Require infrastructure values and secrets; default only listen address, port, and log level. Require upstream hosts and ports rather than defaulting to `localhost`.
32. Give each configuration an `init(config:)` extension in the executable's `Configuration` folder. Configure key material by path and open the file there, failing loudly naming both the key and the path.
33. Bootstrap logging inline in each composition root: stdout plus in-process shipping to the log aggregator, with the service name as a label. No shared logging module.

### Identity and access

34. Put the claim payload, signer, verifier, and one product per transport in a shared `<project>-identity` package. Sign with an asymmetric key; the authenticating service alone holds the private key. Never a shared HMAC secret.
35. Carry `sub`, `role`, `iss`, `iat`, `exp` and nothing else. Permissions, scopes, and entitlements stay in the service that enforces them. Keep the payload's initializer internal so the signer is its only source, and have the signer return the identity it made.
36. Identify a caller and require one as separate steps. Identification passes an absent token through anonymously and refuses a token that is present but invalid; requiring is the handler's per-RPC decision. Absent is `unauthenticated`; insufficient is `permissionDenied`.
37. Exclude the RPCs and routes that issue a session — login, refresh, service-token exchange — from identification entirely.
38. Propagate a caller by forwarding the token it arrived with, bound alongside the identity in a task local. Register the forwarding interceptor on the client, never reissue a token, and let a call made outside any request go out unauthenticated — unless the process holds its own identity.
39. Give a process that calls as itself the `service` role, never `admin`. Its subject is a credential name, so every person-only guard tests the role before parsing the subject. Which RPCs admit a process is each service's per-RPC decision; the HTTP gateway refuses `service` on every route that sends the subject upstream as a user id.
40. Issue a service credential with an operator command on the authenticating service, never an RPC; store only its bcrypt digest; exchange it through a public `IssueServiceToken` RPC that answers an unknown id and a wrong secret identically and returns a token with its own lifetime and no refresh token. Exchange once at startup, racing the service group. One client per identity.
41. Parse `authorization` yourself: scheme case-insensitively, first entry only, `replaceOrAddString` when attaching, the same parser on every transport.

### Transport security

42. Mutually authenticate every internal gRPC connection — service to service and every client of the workflow engine — with a private CA the stack issues itself at first start. There is no plaintext mode and no mode variable.
43. Each process gets one leaf whose SAN is its service name, presents it in both directions, mounts only its own directory, and refuses to start without it. Configure the material by path through two `static func mTLS(config:)` factories on grpc-swift's own transport-security types, one per direction, in each service's `Configuration` folder.
44. The stack's CA reaches only the stack. A client of a managed workflow engine or any external endpoint uses TLS with the system trust roots and that provider's credential, never the internal leaf or CA.

### Durable workflows

45. Keep Temporal SDK code in `<Service>Workflows`. Keep workflow-client protocols, states, results, and Activity service ports in Core without importing Temporal.
46. Keep Workflow code deterministic and side-effect free. Put every external effect in a retry-safe Activity with one side effect each; derive idempotency keys from immutable Workflow input, never a UUID minted in Workflow code.
47. Commit local database work before starting or signaling a Workflow. Never hold a transaction across a Temporal operation. Do not add reconciliation loops, signal-with-start, or workflow-resume-from-database unless the user explicitly requires them.
48. Use deterministic workflow IDs, `.rejectDuplicate` reuse, `.useExisting` conflicts, signal-only commands, and queries for observation. Do not wait for completion from a signal command.
49. A worker owns no database, and its manifest says so: `<Service>Worker` links Core, GRPC, Workflows, the provider adapters its Activities use, and `ServiceIdentity` — never `<Service>Postgres` or a database driver. It reaches its service's data over gRPC as a `service`, through worker-facing RPCs the service exposes in a separate gRPC service beside the user-facing one. Deploy the worker image at the same tag as the service.
50. Give an Activity a distinct registration name whenever two containers on one worker would otherwise share it.

### API gateway

51. A gateway is not a service. Name the package `<organization>-api` with exactly two targets — `API` and `<Project>` — and no Core, Postgres, or provider target. Keep `openapi.yaml` beside its generator configuration in the `API` target and generate `types` only.
52. Chain request contexts `BasicRequestContext` → `IdentityRequestContext` → `AdminRequestContext`, carrying `coreContext` across rather than rebuilding it, so path parameters survive.
53. Register routes in three tiers: session-issuing routes outside the authenticating middleware, an identifying tier that admits anonymous requests, and a requiring tier. Never a path exception inside a middleware.
54. Give administrative routes the same path as the resource they act on, gated by a context conversion at the verb, never an `/admin` prefix. Keep verb-shaped routes verb-shaped.
55. Map `RPCError` to HTTP by code and answer every failure as RFC 9457 problem details. Never collapse upstream failures into 500. Make schema conversions throw on a malformed upstream value rather than drop it.

### Environment

56. Keep every environment concern in environment.md alone. Other references say only that a value "comes from the environment".
57. Publish nothing that nothing outside the stack calls. The gateway gets its address from a sidecar or the platform ingress, not a host port; internal services are reached by name.
58. Mount secrets as files; guard required variables with `${VAR:?message}`; verify every anchor is referenced; render with `docker compose config` before `up`.
59. The first start is staged: infrastructure and the issuer's migration, issue service credentials, then everything.

### Working in an existing system

60. Preserve behavior, naming, and unrelated user changes. Make only the boundary-required adjustment when copying code into a new service.
61. Never infer permission to drop tables, discard data, or delete a migration. Finish the non-destructive work and surface the decision.
62. For greenfield work, state consequential assumptions and choose the smallest architecture that satisfies current requirements.

## Choose the workflow

- One new service: **Build a service**.
- A new distributed system: **Design a system**, then **Build a service** per bounded context, then **Build an API gateway**.
- An existing modular monolith: **Extract a bounded context** with the runbook.
- A focused change: load only the relevant references and preserve the established architecture.

### Design a system

1. Capture capabilities, actors, scale, latency, availability, security, compliance, and consistency requirements.
2. Define bounded contexts and data ownership before repositories or RPCs. Produce a service map with callers, dependencies, and failure boundaries; reject cycles.
3. Choose a local call, synchronous gRPC, an event, or a durable workflow per interaction. Document why each remote boundary exists.
4. Define versioned contracts and failure semantics before implementing producers and consumers.
5. Create and tag the shared proto and identity packages, then initialize each service.
6. Build vertical capabilities through Core, Postgres, gRPC, composition, and environment — not empty horizontal layers across every service.
7. Add deadlines, idempotency, retries, observability, health behavior, and security in proportion to the requirements.

### Build a service

1. Run `swift package init --type executable`, then reshape to `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, `<Service>`, plus `<Service>Workflows` and provider targets only when needed.
2. Implement feature-first Core: entities, commands, repositories, context protocols, use-case contracts, typed errors, policies, use cases.
3. Implement Postgres: context, database, statements, repositories, error translation, `CreateServiceRole` first, then ordered migrations; policies where rows belong to callers.
4. Implement generated gRPC service conformances and feature-local `Protobuf/` conversions, with identity checks per RPC.
5. With Temporal: Workflows, Activities, the Core workflow-client port, its Temporal adapter, and the `<Service>Worker` executable with its own composition root.
6. Add `serve` and `database migrate` composition roots with configuration, logging, transport-security factories, and lifecycle.
7. Add the service's environment pieces per environment.md.
8. Build the whole package, then exercise contracts, identity paths, and infrastructure boundaries.

### Build an API gateway

1. Run `swift package init --type executable`, then reshape to `API` and `<Project>`.
2. Put `openapi.yaml` and `openapi-generator-config.yaml` in `Sources/API/`, generating `types` with `package` access.
3. Implement the request-context chain, the error middleware, and the problem types.
4. Implement one controller per resource, holding generated client protocols, with registration methods named for the tier they mount into.
5. Implement `Schemas/Requests` and `Schemas/Responses` conversions that throw on a malformed upstream value.
6. Add the `serve` composition root: verifier, one client per upstream with required host and port, the three router tiers, the application, one `ServiceGroup`.
7. Add the gateway's environment pieces per environment.md.
8. Build, then exercise the tiers: a session-issuing route must reach its handler while carrying an unverifiable bearer token, and a protected route carrying the same token must be refused.

### Extract a bounded context

1. Inspect repository instructions, dirty state, package graph, callers, persistence, migrations, jobs, configuration, and deployment.
2. Trace the context end to end; identify cross-context reads, transactions, shared types, and compatibility seams.
3. Define and release the contract, build the producer as a standalone service, then keep the consumer's use-case protocol and replace its implementation with a gRPC client.
4. Construct one long-lived client in the consumer composition root and run it in `ServiceGroup`.
5. Deploy the database, migration job, producer, and consumer in dependency order; move and reconcile data explicitly.
6. Remove old persistence only after cutover and rollback criteria pass.

## Completion gates

Do not call work complete until every applicable gate passes.

**Build and boundaries**
- Every package builds in Swift 6 language mode.
- Generated protobuf types appear only in proto, transport, and client-boundary code; Postgres types only in Postgres and composition code.
- Single-operation use cases use `withConnection`; multi-operation units use `withTransaction`; an RLS service has no `withConnection`.

**Persistence**
- Every service owns a dedicated database; migrations are applied; no service-named schema exists.
- Every entity identifier is database-generated and absent from create inputs.
- Every retryable mutation has owner-enforced idempotency with explicit same-key/different-input conflict behavior.
- The first migration is `CreateServiceRole`, and every command but `database migrate` connects as that role.
- An RLS service stamps every transaction and its policies were exercised as the service role with a user, another user, an administrator, and a process.

**Contracts and transport**
- Producers map typed failures to stable status codes; consumers map them to local contracts.
- Contracts are versioned and additive; canonical protos are not duplicated.
- Remote calls have explicit failure semantics; retries are limited to idempotent operations.

**Identity and security**
- Exactly one service holds the signing key; every other service holds only the public key, by path, mounted.
- Session-issuing RPCs and routes are excluded from identification; a client presenting an expired token can still refresh.
- A forwarded token reaches the next service unchanged; every transport parses the credential the same way.
- Every gRPC server refuses a client without a certificate the stack's CA signed; every client verifies the server's name; every process missing its own certificate fails at startup naming the path.
- Every process that calls as itself holds a `service` credential issued by an operator command and exchanged at startup; every person-only guard tests the role before parsing the subject.

**Workflows**
- Temporal clients and workers are lifecycle-managed; Workflows contain no side effects; Activities are retry-safe and distinctly named; the worker is its own executable whose target links no database.

**Gateway**
- Two targets, no database, OpenAPI document in the generating target.
- Child contexts preserve path parameters; administrative routes share the resource path and are gated by context conversion.
- Upstream failures reach the client as problem details with the mapped status.

**Operations**
- Every long-lived client, server, and worker is in a `ServiceGroup` with graceful shutdown.
- Logs carry a service label and searchable identifiers and reach the aggregator; secrets never appear in logs or environment variables.
- Nothing is published that nothing outside the stack calls; `docker compose config` renders every required value; a stack missing a key fails at `up`.
- Extraction work leaves no retired direct database access after cutover.

If a gate requires an unresolved product, consistency, security, or data-migration decision, stop at the safe boundary and request that decision rather than inventing behavior.

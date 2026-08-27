---
name: swift-grpc-microservices-skill
description: Design, build, deploy, and evolve production Swift gRPC microservices and distributed systems using strict reusable module and code conventions. Use when creating a service or multi-service system from scratch, defining service boundaries and communication patterns, structuring SwiftPM packages, designing versioned protobuf contracts, implementing Postgres ownership and transactions, wiring grpc-swift clients or servers, coordinating durable Temporal workflows and workers, preserving XUseCase/XRepository/XCommand boundaries, adding composition roots and Docker Compose deployment, or extracting bounded contexts from a modular monolith.
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
- Read [api-gateway.md](references/api-gateway.md) when building or changing the HTTP surface in front of the system: Hummingbird routing, request contexts, controllers, OpenAPI types, or HTTP-to-RPC translation.
- Read [identity-and-access.md](references/identity-and-access.md) when a caller must be identified or a capability protected: signing keys, token issuance, transport interceptors or middleware, propagating a caller between services, or a process — a worker, a webhook handler — that must call other services as itself.
- Read [distributed-systems.md](references/distributed-systems.md) for every greenfield multi-service system and every task involving cross-service communication, consistency, resilience, security, or observability.
- Read [temporal-workflows.md](references/temporal-workflows.md) in full when adding or changing Temporal workflows, Activities, clients, signals, queries, timers, or workers.
- Read [extraction-runbook.md](references/extraction-runbook.md) in full when extracting a bounded context or decomposing an existing monolith.

## Follow the governing rules

1. Name service targets `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`. Add `<Service>Workflows` only when the service uses Temporal, and one `<Service><Technology>` leaf target per third-party provider SDK; never substitute `Domain`, `Application`, or `Infrastructure`.
2. Use `XUseCase`, `XUseCaseProtocol`, `XUseCaseInput`, `XUseCaseError`, `XUseCaseContext`, `XRepository`, `XRepositoryError`, and repository-facing `XCommand` names.
3. Keep Core independently buildable. It may link a focused library such as `swift-crypto` or a shared contract package, but never an infrastructure SDK: Postgres, gRPC, protobuf, logging, configuration, mail or payment providers, or server libraries. A third-party SDK is what earns an adapter target; conforming to a Core protocol does not.
4. Keep the `Database` protocol in Core. Use `withConnection` for a single repository operation and `withTransaction` only for a multi-operation atomic use case — except in a service that confines callers with row-level security, whose protocol declares `withTransaction` alone because the caller's identity is stamped on the transaction and a read outside one sees no rows. Do not hold a database transaction across an RPC; the sole exception is rotating a single-use secret, where a post-commit failure would destroy a valid credential, and only with an explicit deadline on the call.
5. Let repositories or the database stamp persistence-owned dates. Use a database default such as `DEFAULT NOW()` in Postgres. Do not inject a `now` closure into a use case.
6. Start with primitive domain values when sufficient. Introduce value objects only for established invariants or behavior, not architectural ceremony. Keep business rules in Core as plain structs named `XPolicy` or `XValidator` — never protocols, never injected — and name a duration `expiration`. Configuration is deployment wiring; a policy is a product decision the composition root translates configuration into. Do not inject a concrete struct that has no protocol.
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
17. Keep Temporal SDK code in `<Service>Workflows`. Keep workflow-facing protocols, states, results, and Activity service ports in Core without importing Temporal.
18. Keep Workflow code deterministic and side-effect free. Put database, network, email, and cross-service work in retry-safe Activities.
19. Commit PostgreSQL work before starting or signaling a workflow. Never hold a transaction across a Temporal operation, and do not add a periodic reconciliation loop, signal-with-start, or persisted workflow-resume logic unless the user explicitly requires it.
20. Use deterministic workflow IDs, `.rejectDuplicate` reuse, `.useExisting` conflicts, signal-only commands, and queries for observation. Do not wait for workflow completion from a signal command.
21. Generate every persistent entity identifier in the database that owns the entity, using `UUID DEFAULT uuidv7()`. Omit that identifier from create commands and create RPC requests; return it from the owner after insertion. Never preallocate or generate another service's identifier.
22. Model non-identifier secrets such as verification and refresh tokens as one Core value type that mints and digests, not a generator protocol. Persist only the SHA-256 digest, hand the value to its bearer once, give the column no generation default, and delete the row at terminal state. Reserve bcrypt for user-chosen passwords.
23. Make every remotely retryable mutation idempotent in the service that owns the side effect. Use a stable, namespaced operation key, enforce it atomically with persistence or the provider, return the prior outcome for the same canonical input, and reject reuse with different input. Never use an idempotency key as authorization or as an entity identifier.
24. Guard every state transition in the `WHERE` clause and treat the absent row as the lost race. Carry only the destination state in the command and derive its one legal predecessor from the state enum.
25. Translate an error only where the translation carries information. Never funnel every failure of an adapter into a single-case enum; let the infrastructure error propagate and classify it where the distinction is actionable. Name a repository error for the constraint that exists.
26. Depend on organization packages by tagged URL, never `.package(path:)`. Publish and tag the shared package before a service pins it.
27. Put the claim payload, signer, verifier, and one product per transport in a shared `<project>-identity` package, and sign with an asymmetric key. The service that authenticates users holds the private key and is the only issuer; every other service gets the public key. Never use a shared HMAC secret, which makes every verifier a forger. Keep the payload's memberwise initializer internal so the signer is its only source, let the signer own `iss` and `iat`, and have it return the identity it made so the caller reports the expiry it signed rather than reading the clock twice.
28. Identify a caller and require one as separate interceptors. Identification passes an untokened call through anonymously and refuses a token that is present but does not verify; enforcement is applied only to protected RPCs. Exclude the RPCs that issue a session — password login and token refresh — from identification entirely, or a client that attaches its expired token to every call is refused the only call that could replace it.
29. Propagate a caller by forwarding the token it arrived with, bound alongside the identity in a task local because `ServerContext` carries no metadata. Register the forwarding interceptor on the client rather than per call, never reissue a token, and let a call made outside any caller's request go out unauthenticated — unless the process holds a service identity, in which case it presents that through a separate client (rule 38).
30. Configure key material by path and open the file in the composition root, not the library. A path is the form NIOSSL and grpc-swift already take credentials in — `TLSConfig.CertificateSource.file(path:format:)`, `NIOSSLCertificate.fromPEMFile` — so TLS material configures identically, and it keeps a private key out of the environment, where it is readable from `/proc/<pid>/environ`, reported by the runtime's inspect command, and inherited by every child process. Mount each key as a deployment secret, fail loudly naming both the key and the path, and generate the pair with a script that writes PEM files under a `0700` directory with `umask 077` set before `openssl` writes. Merge the verification anchor into every service that verifies, and confirm it renders: an unreferenced YAML anchor is ignored, so its `${VAR:?message}` guard never fires. Where a system still carries the encoded document instead, test the input for a PEM header before attempting base64 — decoding tolerates every character a PEM contains and silently produces rubbish.
31. A gateway is not a service. Name the package `<organization>-api`, give it exactly two targets — `API` for the surface and `<Project>` for the composition root — and add no Core, Postgres, or provider target. Keep `openapi.yaml` in the `API` target directory beside its generator configuration and generate `types` only; the gateway registers its own routes, so generated server stubs are a second routing mechanism.
32. Chain gateway request contexts `BasicRequestContext` → `IdentityRequestContext` → `AdminRequestContext`, and carry `coreContext` across rather than rebuilding it from the parent. `CoreRequestContextStorage.init(source:)` starts `parameters` and `endpointPath` empty, so rebuilding silently discards path parameters already extracted and a route declared with `:id` matches and then cannot find it.
33. Register HTTP routes in three tiers: session-issuing routes outside the authenticating middleware, an identifying tier that still admits an anonymous request, and a requiring tier. This is the gRPC session-RPC exclusion restated one transport over — token refresh carries a refresh token in the `Authorization` header, so an authenticating middleware verifies it as a claim payload, fails, and refuses the route before its handler runs. Group by tier; never add a path exception inside the middleware.
34. Give administrative routes the same path as the resource they act on and separate them with a request-context conversion at the verb, never an `/admin` path prefix. Keep verb-shaped routes verb-shaped: sessions, provider webhooks, and checkout callbacks are workflows, not resources. Where two operations would share a method and path but differ in scope, keep them distinct rather than collapsing them into one operation with two meanings.
35. Map `RPCError` to HTTP by code and answer every failure as RFC 9457 problem details. Never collapse upstream failures into 500 — the producing service already classified the failure, and discarding that at the last hop tells a client to retry what cannot succeed.
36. Publish only what is called from outside the stack. Databases and orchestrators get no `ports:`; make each published host port a variable, run suite compose files from registry images, and declare required secrets as `${VAR:?message}` beside a `.env.example`.
37. Give a process that calls other services as itself the `service` role, never `admin`: an admin token works at the HTTP edge too. Its subject is a credential name, so a person-only guard tests the role before it parses the subject — the shared `IdentityContext.require…Identity()` checks in the gRPC product do this once; a handler calls one and keeps only the check that is its own — a credential can be issued under any name, including a UUID — and the gateway refuses `service` with `requireUser()` on every route that sends the subject upstream as a user id. Which RPCs admit a process is each service's per-RPC decision, like the admin check. Bump every verifying service to the identity tag that decodes the new role before the first such token is minted.
38. Issue a process its credential with an operator command on the authenticating service, never an RPC: a name, a minted secret printed once, only the bcrypt digest stored in the issuer's database. Exchange it through a public `IssueServiceToken` RPC that answers an unknown id and a wrong secret identically, returning a token with its own lifetime and no refresh token — the expiry is the revocation delay. On the consumer, keep the session, the interceptor, and the `IssueServiceToken` adapter together in a `ServiceIdentity` product of the identity package — the one product there that links the authentication contract — and never in the issuer's own package. One client per identity: a client carrying the service interceptor speaks as the process on every call, and a process that also forwards callers uses a second client. Exchange once at startup, racing the service group, so a bad credential fails there.
39. Make `CreateServiceRole` the first migration of every service: `CREATE ROLE <service>_service LOGIN PASSWORD`, `GRANT CONNECT`, schema usage, DML on all tables and sequences, and the same as default privileges — plain statements, nothing more. Migrations run as the owner; `serve`, `worker`, and every other command connect as the service role. Do this whether or not the service has row-level security, because the owner is exempt from every policy and the migration library refuses a reordered list, so a role added later cannot be made first without re-migrating.
40. Confine caller-owned rows with Postgres row-level security, and state the rule once: policies in migrations, never a scope restated in every statement. The policies are enforced against the service role of rule 39. Stamp each transaction with `set_config('app.caller_role', …, true)` and, for a person only, `app.caller_user_id` — bound parameters, transaction-scoped, named for the caller because a caller need not be a user. Keep `NULLIF(current_setting(…, true), '')::uuid` in every policy, admit `service` beside `admin`, and verify as the service role: as the owner every test passes with the policies off. The handler still answers what a policy cannot — `permissionDenied` for a named user who is not the caller, and a process on a person-only RPC.

## Choose the workflow

- For one new service, follow **Build a service**.
- For a new distributed system, follow **Design and build a system**, then repeat **Build a service** for each bounded context.
- For an existing modular monolith, follow **Extract an existing bounded context** and the extraction runbook.
- For the HTTP surface in front of a system, follow **Build an API gateway**.
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
2. Reshape the package to `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`. Add `<Service>Workflows` only when durable orchestration is required.
3. Implement feature-first Core entities, commands, repositories, context protocols, use-case contracts, typed errors, and use cases.
4. Implement Postgres database/context, prepared statements, repositories, constraints, error translation, and ordered migrations.
5. Implement generated gRPC service protocols and feature-local `Protobuf/` conversion directories.
6. When Temporal is required, implement Workflows, Activities, a Core workflow-client port, a Temporal client adapter, and the executable `worker` command.
7. Add the `serve` and `database migrate` command composition roots, configuration, logging, and lifecycle management.
8. Add `.env.example`, container build support, and Compose services for Postgres, migration, and the service.
9. Build the complete package, then exercise contracts and infrastructure boundaries.

## Build an API gateway

1. Run `swift package init --type executable`, then reshape the package to `API` and `<Project>`.
2. Put `openapi.yaml` and `openapi-generator-config.yaml` in `Sources/API/`, generating `types` with `package` access.
3. Implement the request-context chain, the error middleware, and the RFC 9457 problem types.
4. Implement one controller per resource, holding generated client protocols and exposing registration methods named for the tier they mount into.
5. Implement `Schemas/Requests` and `Schemas/Responses` conversions that throw on a malformed upstream value rather than dropping it.
6. Add the `serve` composition root: verifier, one client per upstream with required host and port, the three router tiers, the application, and one `ServiceGroup`.
7. Add `.env.example`, container build support, and a Compose service publishing only the HTTP port.
8. Build, then exercise the tiers directly: a session-issuing route must reach its handler while carrying an unverifiable bearer token, and a protected route carrying the same token must be refused.

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
- Temporal clients and workers are lifecycle-managed; Workflows contain no external side effects, and retryable Activities are idempotent.
- Every service owns a dedicated database with applied migrations and no service-specific schema.
- Every persistent entity identifier is generated by its owning database and is absent from caller-supplied create inputs.
- Every retryable mutation and Temporal write Activity has receiver-enforced idempotency and explicit same-key/different-input conflict behavior.
- Remote calls have explicit failure semantics; retries are limited to safe/idempotent operations.
- Contracts are versioned and compatibility-safe; canonical protos are not duplicated.
- Exactly one service holds a signing key; every other service holds only the verifying key.
- Session-issuing RPCs are excluded from caller identification, and a client presenting an expired token can still refresh.
- Every transport reads the credential the same way, and a forwarded token reaches the next service unchanged.
- Every service that verifies a token receives both the key path and the mounted file in its rendered configuration, and a stack missing a key fails to start rather than at the first request.
- A process that calls as itself holds a `service` credential issued by an operator command, exchanges it at startup, and every person-only guard it could reach tests the role before parsing the subject; every verifying service is on an identity tag that decodes the role.
- Every service's first migration is `CreateServiceRole`, and every command but `database migrate` connects as that role.
- A service with row-level security stamps every transaction with the caller's role and, for a person, user id, and its policies were exercised as that role with a user, another user, an administrator, and a process.
- A gateway package has exactly two targets, owns no database, and keeps its OpenAPI document in the target that generates from it.
- Child request contexts preserve path parameters; a route declared with a parameter resolves it inside the child.
- Session-issuing HTTP routes sit outside the authenticating middleware, and a refresh presenting an unverifiable bearer token reaches its handler rather than a 401.
- Administrative routes share the path of the resource they act on and are gated by a context conversion, not a path prefix.
- Upstream failures reach the client as problem details carrying the mapped status, not a blanket 500.
- Logging, metrics/tracing strategy, health behavior, secrets, and network exposure match the operating environment.
- Deployment ordering and representative dependency failures are verified.
- Extraction work leaves no retired direct database access after cutover.

If a gate requires an unresolved product, consistency, security, or data-migration decision, stop at the safe boundary and request that decision rather than inventing behavior.

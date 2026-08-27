# Modular-monolith extraction runbook

Use this runbook for one bounded context and repeat it for an entire monolith. Work in vertical slices; do not attempt a mechanical target-per-layer rewrite of the whole system.

## Contents

- Establish the baseline and choose extraction order
- Define and release the contract
- Build the producer and adapt the consumer
- Wire deployment, move data, and cut over
- Retire the old slice and verify
- Progress through the whole monolith

## 1. Establish the baseline

1. Read repository instructions and inspect `git status` in every involved repository.
2. Build the relevant packages before editing. Record pre-existing failures rather than fixing unrelated code.
3. Inventory `Package.swift`, products, targets, direct dependency edges, executables, command trees, configuration extensions, deployment manifests, and CI/build commands.
4. Trace each selected feature from inbound controller/handler through use-case protocol and implementation to repositories, statements, migrations, tables, and background jobs.
5. Search by entity, repository, table, schema, use-case protocol, configuration key, and operational file.

Produce a boundary inventory:

| Item | New service | Consumer/monolith | Temporary bridge |
| --- | --- | --- | --- |
| Business behavior | Core use cases | Caller-facing protocols only | gRPC-backed implementations |
| Data | Dedicated service database | No direct access after cutover | Explicit data migration |
| Contract | Versioned proto | Released proto dependency | Compatible `v1` rollout |
| Runtime | gRPC server and migrations | Long-lived gRPC client | Internal DNS |

## 2. Choose extraction order

Draw the feature dependency graph. Extract leaf capabilities with clear ownership first. For every incoming or outgoing dependency, classify it as:

- local domain behavior that moves with the slice;
- synchronous query/command requiring an RPC;
- asynchronous business fact that may later require messaging;
- accidental shared persistence that must be severed;
- shared utility that can remain a small package dependency.

Do not extract two contexts into one service solely because their tables currently join. Decide the owning service and replace the join with an explicit contract or a locally maintained projection based on actual consistency requirements.

## 3. Define and release the contract

1. Add `<Service>Protos` to the shared `<project>-protos` package.
2. Put files under `Sources/<Service>Protos/<organization>/<service>/v1/`.
3. Define messages around service capabilities, not database tables. Use RPC names matching business operations.
4. Include only fields consumers need. Model stable identifiers explicitly.
5. Generate public clients, servers, and messages.
6. Build the proto package, review generated symbol names, commit, tag, and publish a semantic version.

Prefer additive evolution. Reserve removed fields and names. Do not modify a released message incompatibly.

## 4. Build the producer package

1. Run `swift package init --type executable` in the new repository.
2. Reshape targets to `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, and `<Service>`.
3. Copy the selected entities, repository ports/errors/commands, use-case contracts, and behavior into feature-first Core folders.
4. Remove monolith framework imports from Core. Preserve behavior and naming; do not introduce new value objects or clock abstractions during extraction.
5. Reintroduce the Core `Database` protocol and use-case context protocols.
6. Copy and adapt Postgres repositories, statements, and migrations. Remove the old service schema because the new service owns the database. Let persistence stamp creation dates.
7. Add gRPC services and colocated `+Protobuf` mappings.
8. Add `serve` and `database migrate` composition roots.
9. Add the service's environment pieces per [environment.md](environment.md).
10. Build the entire producer.

When copying as-is conflicts with a settled convention, make only the boundary-required adjustment. Examples: convert `public` to `package`, change a schema-qualified table to an unqualified table, or replace an old shared monolith database wrapper with the Core `Database` protocol.

## 5. Adapt the monolith consumer

1. Add the released proto dependency.
2. Add direct gRPC/protobuf products only to targets that import them.
3. Keep the existing API-facing `XUseCaseProtocol`, input, error, and entity so callers do not absorb transport types.
4. Replace the concrete local use-case implementation with one that wraps the generated gRPC client.
5. Map local input to requests, responses to local entities, and RPC status to local use-case errors.
6. Construct the shared `GRPCClient` in the executable composition root from scoped required host/port configuration.
7. Inject it into the consumer use cases and include it in `ServiceGroup`.

Do not leave the old Postgres database/context variable in composition once no local use case for the extracted feature consumes it. Remove obsolete imports and manifest dependencies from the affected target.

## 6. Wire deployment

1. Add dedicated service database credentials to `.env.example`.
2. Add `<service>-postgres` with Postgres 18 and `/var/lib/postgresql` volume mounting.
3. Add `<service>-migrate`, gated on the database health check.
4. Add `<service>`, gated on successful migration and available only on the internal network.
5. Add consumer `GRPC_<SERVICE>_HOST` and `_PORT` using the producer's name on the internal network.
6. Add a consumer dependency on the service for startup ordering.
7. Render and validate the environment ([environment.md](environment.md)) and verify image names and commands.

On a managed container platform or any other ingress, expose only the public HTTP edge. Keep gRPC service-to-service traffic private. Use platform TLS at the edge; revisit mTLS only when the threat model requires it.

## 7. Move data and cut over

Treat data migration as a separate explicit plan:

1. Measure existing rows and constraints.
2. Choose downtime copy, dual-write, change-data capture, or backfill plus short cutover based on availability requirements.
3. Define idempotent import behavior and reconciliation checks.
4. Back up before destructive operations.
5. Deploy the producer and verify health/migrations.
6. Move and reconcile data.
7. Deploy the consumer using gRPC.
8. Observe errors, latency, and row counts.
9. Keep rollback viable until confidence criteria pass.

Never infer permission to drop old tables or discard data. If the user has not chosen the migration strategy, finish the non-destructive extraction and surface the decision.

## 8. Retire the old slice

After cutover verification:

- Remove old Postgres repositories, statements, contexts, migrations, and schema creation for the extracted context.
- Remove obsolete database construction from the monolith composition root.
- Remove unused package dependencies and imports.
- Search for old table/schema names and repository types across source, scripts, and deployment.
- Confirm no other context directly reads the extracted database.

## 9. Verification matrix

Run the smallest checks early and full checks at gates:

| Repository | Required checks |
| --- | --- |
| Proto package | `swift build`; inspect target/product and versioned path |
| Producer | `swift build`, migration registration, executable help/command tree |
| Consumer | full build; search for direct persistence references |
| Deployment | Environment render, database health gate, migration completion gate, internal DNS/env alignment |

Also inspect the final diff and dirty state in every repository. Do not claim failures caused by pre-existing changes were introduced by the extraction.

## 10. Whole-monolith progression

Maintain an extraction ledger for all contexts:

```text
candidate → contract designed → producer ready → consumer adapted → deployed → data cut over → local slice retired
```

Complete one context through retirement before multiplying partial extractions, unless a shared contract release or deployment change makes parallel preparation clearly safer. Reassess dependency direction after every extraction because removing one context changes the remaining monolith's seams.

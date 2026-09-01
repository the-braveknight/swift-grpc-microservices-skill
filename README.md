# Swift gRPC Microservices Skill

An agent skill that teaches Claude Code, Codex, and other AI agents to design and build production Swift gRPC microservices and distributed systems a particular way — one set of principles, names, boundaries, and operational habits learned from running such systems, rather than a survey of options.

The skill standardizes:

- SwiftPM module names, dependency direction, and target boundaries — `<Service>Core`, `<Service>Postgres`, `<Service>GRPC`, `<Service>`, plus `<Service>Workflows` and one leaf target per provider SDK
- feature-first Core with `XUseCase`, `XRepository`, `XCommand`, and `XPolicy` conventions, typed errors, and domain-event logging through the `swift-log` facade
- Postgres ownership: database-generated UUIDv7 identifiers, noun-named dates, prepared statements, guarded state transitions, partial unique indexes, idempotent writes, a service role created by the first migration, and row-level security stamped per transaction
- versioned protobuf contracts in one shared package and grpc-swift producer/consumer boundaries
- executable composition roots with ServiceLifecycle, scoped configuration, inline logging bootstrap, and in-process log shipping
- a shared identity package: EdDSA single-issuer tokens, identification separated from enforcement, caller propagation, and a `service` role for processes with operator-issued credentials
- mutual TLS on every internal connection from a CA the stack issues itself, with all key material configured by path
- a Hummingbird API gateway with layered request contexts, tiered routing, flat admin routes, and RFC 9457 error translation
- durable Temporal workflows, Activities, signals, queries, and a worker that is its own executable and owns no database
- service boundaries, communication choices, consistency, resilience, and observability for a whole system
- one environment reference — images, Compose, ports, secrets, certificates, the gateway's address, the staged first start — kept out of every other file
- infrastructure-free use-case tests: a `<Service>CoreTests` target with actor mocks, a scoped database helper, and a guard/translation/success matrix
- branch-per-environment delivery: per-commit SHA-tagged images from a Containerfile, a reused swiftlang test workflow, shared deployment actions in one `ci` repository, migrations awaited before serve rolls
- staged extraction of bounded contexts from a modular monolith

The agent instructions are in [SKILL.md](SKILL.md): principles, themed governing rules, workflows, and completion gates. Detail lives in [references](references), one file per topic, loaded as needed.

## Install

Claude Code:

```bash
git clone https://github.com/the-braveknight/swift-grpc-microservices-skill.git \
  ~/.claude/skills/swift-grpc-microservices-skill
```

Codex:

```bash
git clone https://github.com/the-braveknight/swift-grpc-microservices-skill.git \
  ~/.codex/skills/swift-grpc-microservices-skill
```

## Use

In Claude Code, invoke it with `/swift-grpc-microservices-skill`, or simply describe a matching task — the skill description triggers automatic loading. In Codex:

```text
Use $swift-grpc-microservices-skill to design and build a production-ready distributed ordering system in Swift.
```

The skill covers a single service, a whole system, the HTTP gateway in front of it, protobuf contracts, Postgres ownership, identity and transport security, Temporal orchestration, the Compose environment, and monolith decomposition.

## Adapt

The skill is generic: `<organization>`, `<project>`, and `<service>` are placeholders. Its defaults for tooling — Grafana Loki for logs, a Tailscale sidecar for the gateway's address, `step` for the internal CA — are choices, and each is confined to one section so a different tool changes one place. The principles and rules do not depend on those choices.

## Maintain

The governing rules in [SKILL.md](SKILL.md) restate reference content in compressed form so agents can follow them without loading every file. When editing a reference, re-check the corresponding rules and completion gates for drift, and vice versa; the references are the source of truth for detail, the rules for scope. Each topic has one home — cross-reference by section name, never by rule number.

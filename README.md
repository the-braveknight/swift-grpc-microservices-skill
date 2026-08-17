# Swift gRPC Microservices Skill

An agent skill for designing and building production Swift gRPC microservices and distributed systems from scratch, as well as extracting bounded contexts from modular monoliths. Works with Codex and Claude Code.

The skill standardizes:

- SwiftPM module names, dependency direction, and target boundaries
- the supported Swift package dependency and version baseline
- feature-first Core organization with `XUseCase`, `XRepository`, and `XCommand` conventions
- Postgres repositories, prepared statements, transactions, and migrations
- database-owned UUIDv7 identifiers that are omitted from caller create inputs
- versioned protobuf packages and grpc-swift producer/consumer boundaries
- executable composition roots, configuration, lifecycle management, and migrations
- durable Temporal workflows, Activities, signals, queries, clients, and worker composition
- service boundaries, communication choices, consistency, resilience, security, and observability
- PostgreSQL 18 and Docker Compose deployment topology
- staged data migration, rollout, and monolith retirement

The complete agent instructions are in [SKILL.md](SKILL.md). Detailed conventions are loaded from [references](references) as needed.

## Install

For Codex, clone the repository into the Codex skills directory:

```bash
git clone https://github.com/the-braveknight/swift-grpc-microservices-skill.git \
  ~/.codex/skills/swift-grpc-microservices-skill
```

For Claude Code, clone it into the Claude Code skills directory instead:

```bash
git clone https://github.com/the-braveknight/swift-grpc-microservices-skill.git \
  ~/.claude/skills/swift-grpc-microservices-skill
```

## Use

Invoke the skill explicitly in a Codex request:

```text
Use $swift-grpc-microservices-skill to design and build a production-ready distributed ordering system in Swift.
```

In Claude Code, invoke it with `/swift-grpc-microservices-skill`, or simply describe a matching task — the skill description triggers automatic loading.

The skill can also create a single service, define protobuf contracts, implement gRPC integrations, establish Postgres ownership, coordinate durable Temporal workflows, build composition roots, design service communication, or safely decompose an existing monolith.

## Maintain

The governing rules in [SKILL.md](SKILL.md) restate reference content in compressed form so agents can follow them without loading every file. When editing a reference, re-check the corresponding governing rules and completion gates for drift, and vice versa; the references are the source of truth for detail, the rules for scope.

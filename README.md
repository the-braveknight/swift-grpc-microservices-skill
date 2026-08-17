# Swift gRPC Microservices Skill

A Codex skill for designing and building production Swift gRPC microservices and distributed systems from scratch, as well as extracting bounded contexts from modular monoliths.

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

Clone the repository into the Codex skills directory:

```bash
git clone https://github.com/the-braveknight/swift-grpc-microservices-skill.git \
  ~/.codex/skills/swift-grpc-microservices-skill
```

## Use

Invoke the skill explicitly in a Codex request:

```text
Use $swift-grpc-microservices-skill to design and build a production-ready distributed ordering system in Swift.
```

The skill can also create a single service, define protobuf contracts, implement gRPC integrations, establish Postgres ownership, coordinate durable Temporal workflows, build composition roots, design service communication, or safely decompose an existing monolith.

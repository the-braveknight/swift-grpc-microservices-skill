# Service SwiftPM package

## Contents

- Package initialization
- Dependency baseline
- Exact source tree
- Manifest shape

## Package initialization

Initialize the directory first:

```bash
swift package init --type executable
```

Then reshape the generated package. Keep Swift tools 6.3, Swift language mode 6, the current package's platform floor, and dependency versions aligned across the repository set unless the user asks to upgrade.

## Dependency baseline

Use this package set for the architecture in this skill. Treat these versions as the supported baseline, not an instruction to downgrade a repository that already uses compatible newer releases.

| Package | Baseline | Products/purpose |
| --- | --- | --- |
| `swift-argument-parser` | `1.8.2` | `ArgumentParser` for `serve` and `database migrate` commands |
| `swift-configuration` | `1.2.0` | `Configuration` and `EnvironmentVariablesProvider` |
| `swift-service-lifecycle` | `2.11.0` | `ServiceLifecycle` and `ServiceGroup` |
| `swift-log` | `1.15.0` | `Logging` |
| `postgres-migrations` | `1.2.0` | `PostgresMigrations` |
| `postgres-nio` | `1.33.1` | `PostgresNIO`, `PostgresClient`, prepared statements, and transactions |
| `grpc-swift-2` | `2.4.0` | `GRPCCore`, `GRPCClient`, and `GRPCServer` |
| `grpc-swift-nio-transport` | `2.9.1` | `GRPCNIOTransportHTTP2` |
| `grpc-swift-extras` | `2.2.0` | `GRPCServiceLifecycle` adapters |
| `grpc-swift-protobuf` | `2.4.0` | `GRPCProtobuf` and `GRPCProtobufGenerator` |
| `swift-protobuf` | `1.32.0` | `SwiftProtobuf` messages and well-known types |
| shared `<project>-protos` package | first compatible release, such as `0.1.0` | `<Service>Protos` |
| `swift-container-plugin` | `1.3.0` | `build-container-image` command plugin |

Declare them at package level:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.8.2"),
    .package(url: "https://github.com/apple/swift-configuration.git", from: "1.2.0"),
    .package(url: "https://github.com/swift-server/swift-service-lifecycle.git", from: "2.11.0"),
    .package(url: "https://github.com/apple/swift-log.git", from: "1.15.0"),
    .package(url: "https://github.com/hummingbird-project/postgres-migrations.git", from: "1.2.0"),
    .package(url: "https://github.com/vapor/postgres-nio.git", from: "1.33.1"),
    .package(url: "https://github.com/grpc/grpc-swift-2.git", from: "2.4.0"),
    .package(url: "https://github.com/grpc/grpc-swift-nio-transport.git", from: "2.9.1"),
    .package(url: "https://github.com/grpc/grpc-swift-extras.git", from: "2.2.0"),
    .package(url: "https://github.com/grpc/grpc-swift-protobuf.git", from: "2.4.0"),
    .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.32.0"),
    .package(
        url: "https://github.com/<organization>/<project>-protos.git",
        from: "0.1.0"
    ),
    .package(url: "https://github.com/apple/swift-container-plugin.git", from: "1.3.0"),
]
```

Do not add every product to every target. Declare only the direct products imported by that target, using the manifest shape below. The container plugin is invoked from the package command line and does not need to be attached to a source target.

## Exact source tree

```text
Package.swift
Makefile
.env.example
compose.yaml
Sources/
  <Service>/
    <Service>.swift
    Configuration/
      PostgresClient.Configuration+ConfigReader.swift
    Database/
      Database.swift
      Migrate/
        Migrate.swift
    Serve/
      Serve.swift
  <Service>Core/
    Database/
      Database.swift
    <Features>/
      <Entity>.swift
      Repository/
        <Entity>Repository.swift
        <Entity>RepositoryError.swift
        Commands/
          Create<Entity>Command.swift
      UseCases/
        Create<Entity>/
          Create<Entity>UseCase.swift
          Create<Entity>UseCaseContext.swift
          Create<Entity>UseCaseError.swift
          Create<Entity>UseCaseInput.swift
          Create<Entity>UseCaseProtocol.swift
  <Service>Postgres/
    Context/
      Postgres<Service>Context.swift
    Database/
      PostgresContext.swift
      PostgresDatabase.swift
    Migrations/
      <Entity>/
        Create<Entities>Table.swift
    Repositories/
      <Entity>/
        Postgres<Entity>Repository.swift
    Statements/
      <Entity>/
        Create<Entity>Statement.swift
        List<Entities>Statement.swift
  <Service>GRPC/
    <Features>/
      <Entity>Service.swift
      Protobuf/
        <Entity>+Protobuf.swift
        Create<Entity>UseCaseInput+Protobuf.swift
```

Use plural feature folders such as `Items`, then group repository and use-case artifacts within that feature. Do not create top-level `Entities`, `UseCases`, or `Repositories` buckets in Core. In Postgres, group by technical responsibility and then entity because those files implement infrastructure mechanics.

In GRPC, keep the generated-service conformance at the feature root. Put every request/input and entity/message conversion in that feature's single `Protobuf/` directory. Do not split it further into request and response folders.

## Manifest shape

Use this dependency direction:

```swift
targets: [
    .target(name: "<Service>Core"),
    .target(
        name: "<Service>Postgres",
        dependencies: [
            "<Service>Core",
            .product(name: "Logging", package: "swift-log"),
            .product(name: "PostgresMigrations", package: "postgres-migrations"),
            .product(name: "PostgresNIO", package: "postgres-nio"),
        ]
    ),
    .target(
        name: "<Service>GRPC",
        dependencies: [
            "<Service>Core",
            .product(name: "GRPCCore", package: "grpc-swift-2"),
            .product(name: "GRPCProtobuf", package: "grpc-swift-protobuf"),
            .product(name: "SwiftProtobuf", package: "swift-protobuf"),
            .product(name: "<Service>Protos", package: "<project>-protos")
        ]
    ),
    .executableTarget(
        name: "<Service>",
        dependencies: [
            "<Service>Core",
            "<Service>GRPC",
            "<Service>Postgres",
            .product(name: "ArgumentParser", package: "swift-argument-parser"),
            .product(name: "Configuration", package: "swift-configuration"),
            .product(name: "GRPCCore", package: "grpc-swift-2"),
            .product(name: "GRPCNIOTransportHTTP2", package: "grpc-swift-nio-transport"),
            .product(name: "GRPCServiceLifecycle", package: "grpc-swift-extras"),
            .product(name: "Logging", package: "swift-log"),
            .product(name: "PostgresMigrations", package: "postgres-migrations"),
            .product(name: "PostgresNIO", package: "postgres-nio"),
            .product(name: "ServiceLifecycle", package: "swift-service-lifecycle"),
        ]
    ),
],
swiftLanguageModes: [.v6]
```

Include a direct product dependency in every target that imports its module. For example, the executable—not the feature target—needs `GRPCNIOTransportHTTP2` when it constructs the transport. Remove dependencies inherited from the monolith that the extracted slice does not import.

Do not expose internal library products by default. The executable is the package product. Internal targets communicate through `package` declarations.

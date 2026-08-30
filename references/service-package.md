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

Then reshape the generated package. Keep the Swift tools version, Swift language mode 6, the platform floor, and dependency versions aligned across the organization's repositories unless the user asks to upgrade.

## Dependency baseline

These are the packages the architecture is built on. The versions are a floor from when this skill was last revised, not an instruction to downgrade a repository that already uses compatible newer releases; align with the organization's other services first, then with the newest compatible release.

| Package | Baseline | Products/purpose |
| --- | --- | --- |
| `swift-argument-parser` | `1.8.2` | `ArgumentParser` for the command tree |
| `swift-configuration` | `1.2.0` | `Configuration` and `EnvironmentVariablesProvider` |
| `swift-service-lifecycle` | `2.11.0` | `ServiceLifecycle` and `ServiceGroup` |
| `swift-log` | `1.15.0` | `Logging` facade (also linked by `<Service>Core` so use cases log domain events) |
| `swift-log-loki` | `2.0.0` | `LoggingLoki` — in-process log shipping to the aggregator |
| `swift-nio` | `2.65.0` | `NIOFoundationCompat` — declared on any executable that links `LoggingLoki` but not PostgresNIO (the worker, the gateway); swift-log-loki 2.0.0 omits it |
| `postgres-migrations` | `1.2.0` | `PostgresMigrations` |
| `postgres-nio` | `1.33.1` | `PostgresNIO`, `PostgresClient`, prepared statements, transactions |
| `grpc-swift-2` | `2.4.0` | `GRPCCore`, `GRPCClient`, `GRPCServer` |
| `grpc-swift-nio-transport` | `2.9.1` | `GRPCNIOTransportHTTP2` |
| `grpc-swift-extras` | `2.2.0` | `GRPCServiceLifecycle` adapters |
| `grpc-swift-protobuf` | `2.4.0` | `GRPCProtobuf` and `GRPCProtobufGenerator` |
| `swift-protobuf` | `1.32.0` | `SwiftProtobuf` messages and well-known types |
| `swift-temporal-sdk` | `1.0.0` | `Temporal` — only with durable orchestration |
| `hummingbird-auth` | current | `HummingbirdBcrypt` — only in a `<Service>Bcrypt` adapter target |
| `<project>-protos` | first compatible tag | `<Service>Protos` |
| `<project>-identity` | first compatible tag | `Identity`, `IdentityGRPC`, `ServiceIdentity` as needed |
| `swift-container-plugin` | `1.3.0` | `build-container-image` command plugin |

Declare them at package level:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.8.2"),
    .package(url: "https://github.com/apple/swift-configuration.git", from: "1.2.0"),
    .package(url: "https://github.com/swift-server/swift-service-lifecycle.git", from: "2.11.0"),
    .package(url: "https://github.com/apple/swift-log.git", from: "1.15.0"),
    .package(url: "https://github.com/lovetodream/swift-log-loki.git", from: "2.0.0"),
    .package(url: "https://github.com/hummingbird-project/postgres-migrations.git", from: "1.2.0"),
    .package(url: "https://github.com/vapor/postgres-nio.git", from: "1.33.1"),
    .package(url: "https://github.com/grpc/grpc-swift-2.git", from: "2.4.0"),
    .package(url: "https://github.com/grpc/grpc-swift-nio-transport.git", from: "2.9.1"),
    .package(url: "https://github.com/grpc/grpc-swift-extras.git", from: "2.2.0"),
    .package(url: "https://github.com/grpc/grpc-swift-protobuf.git", from: "2.4.0"),
    .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.32.0"),
    .package(url: "https://github.com/apple/swift-temporal-sdk.git", from: "1.0.0"), // only with Temporal
    .package(url: "https://github.com/<organization>/<project>-protos.git", from: "0.1.0"),
    .package(url: "https://github.com/<organization>/<project>-identity.git", from: "0.1.0"),
    .package(url: "https://github.com/apple/swift-container-plugin.git", from: "1.3.0"),
]
```

Do not add every product to every target. Declare only the direct products imported by that target. The container plugin is invoked from the package command line and is not attached to a source target.

Depend on organization packages by tagged URL, never by `.package(path:)`. A path dependency builds only where the sibling repository happens to be checked out, so CI and container builds fail on a package that resolves locally — and a service can silently build against uncommitted contract changes. Publish and tag first, then pin `from:` the release containing what the service imports. Contract additions are additive: tag them as a minor release so consumers on the same major range pick them up without a manifest edit. How to verify a cross-repository change before tagging is in [identity-and-access.md](identity-and-access.md) under *The shared identity package*.

After renaming a target, delete `.build` in that package and every consumer, or the stale `.swiftmodule` keeps the old module name and the compiler insists a module both exists and does not.

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
      TransportSecurity+ConfigReader.swift
      LokiLogProcessorConfiguration+ConfigReader.swift
      IdentityVerifier.Configuration+ConfigReader.swift
      ServiceIdentityCredentials+ConfigReader.swift   # only for a process that calls as itself
    Database/
      Database.swift
      Migrate/
        Migrate.swift
    Serve/
      Serve.swift
  <Service>Worker/                 # only with Temporal — its own executable
    Worker.swift
    Configuration/
      TransportSecurity+ConfigReader.swift            # its own copies of what it reads
      LokiLogProcessorConfiguration+ConfigReader.swift
      ServiceIdentityCredentials+ConfigReader.swift
    ServiceCredentials/            # only on the authenticating service
      ServiceCredentials.swift
      Create.swift
  <Service>Core/
    Database/
      Database.swift
    <Features>/
      <Entity>.swift
      <Rule>Policy.swift
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
      CreateServiceRole.swift
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
  <Service>Workflows/              # only with Temporal
    <Feature>/
      <Feature>Workflow.swift
      <Feature>Activities.swift
      Temporal<Feature>WorkflowClient.swift
  <Service>Bcrypt/                 # one target per provider SDK
    BcryptPasswordHasher.swift
  <Service>Resend/
    Resend<Feature>EmailService.swift
```

Use plural feature folders such as `Items`, then group repository and use-case artifacts within that feature. Do not create top-level `Entities`, `UseCases`, or `Repositories` buckets in Core. In Postgres, group by technical responsibility and then entity because those files implement infrastructure mechanics.

In GRPC, keep the generated-service conformance at the feature root. Put every request/input and entity/message conversion in that feature's single `Protobuf/` directory. Do not split it further.

When Temporal is required, keep its SDK dependency and all macro-decorated Workflows and Activities in `<Service>Workflows`. Keep Core free of Temporal by defining the workflow-client and Activity-service protocols plus workflow state/result values there.

## Manifest shape

```swift
targets: [
    .target(
        name: "<Service>Core",
        dependencies: [
            // The swift-log facade only, so use cases can log domain events.
            // Concrete handlers are wired in the executable, never here.
            .product(name: "Logging", package: "swift-log"),
            .product(name: "Identity", package: "<project>-identity"), // only if Core reads the caller
        ]
    ),
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
            .product(name: "IdentityGRPC", package: "<project>-identity"),
            .product(name: "<Service>Protos", package: "<project>-protos"),
        ]
    ),
    .target(
        name: "<Service>Workflows",
        dependencies: [
            "<Service>Core",
            .product(name: "Temporal", package: "swift-temporal-sdk"),
        ]
    ), // only with Temporal
    .target(
        name: "<Service>Bcrypt",
        dependencies: [
            "<Service>Core",
            .product(name: "HummingbirdBcrypt", package: "hummingbird-auth"),
        ]
    ), // one adapter target per provider SDK
    .executableTarget(
        name: "<Service>",
        dependencies: [
            "<Service>Core",
            "<Service>GRPC",
            "<Service>Postgres",
            "<Service>Workflows", // only with Temporal
            .product(name: "ArgumentParser", package: "swift-argument-parser"),
            .product(name: "Configuration", package: "swift-configuration"),
            .product(name: "GRPCCore", package: "grpc-swift-2"),
            .product(name: "GRPCNIOTransportHTTP2", package: "grpc-swift-nio-transport"),
            .product(name: "GRPCServiceLifecycle", package: "grpc-swift-extras"),
            .product(name: "Identity", package: "<project>-identity"),
            .product(name: "IdentityGRPC", package: "<project>-identity"),
            .product(name: "Logging", package: "swift-log"),
            .product(name: "LoggingLoki", package: "swift-log-loki"),
            .product(name: "PostgresMigrations", package: "postgres-migrations"),
            .product(name: "PostgresNIO", package: "postgres-nio"),
            .product(name: "ServiceLifecycle", package: "swift-service-lifecycle"),
            .product(name: "Temporal", package: "swift-temporal-sdk"), // only with Temporal
        ]
    ),
    // Only with Temporal: the worker is its own executable, so that it links only what it uses.
    .executableTarget(
        name: "<Service>Worker",
        dependencies: [
            "<Service>Core",
            "<Service>GRPC",
            "<Service>Workflows",
            // plus the provider adapters its Activities use — and nothing Postgres
            .product(name: "ArgumentParser", package: "swift-argument-parser"),
            .product(name: "Configuration", package: "swift-configuration"),
            .product(name: "GRPCCore", package: "grpc-swift-2"),
            .product(name: "GRPCNIOTransportHTTP2", package: "grpc-swift-nio-transport"),
            .product(name: "Identity", package: "<project>-identity"),
            .product(name: "IdentityGRPC", package: "<project>-identity"),
            .product(name: "Logging", package: "swift-log"),
            .product(name: "LoggingLoki", package: "swift-log-loki"),
            // swift-log-loki 2.0.0 imports NIOFoundationCompat without declaring it. The service
            // executable gets it transitively through PostgresNIO; this target links no Postgres,
            // so without this line the static link fails on JSONEncoder.encodeAsByteBuffer.
            .product(name: "NIOFoundationCompat", package: "swift-nio"),
            .product(name: "ServiceIdentity", package: "<project>-identity"),
            .product(name: "ServiceLifecycle", package: "swift-service-lifecycle"),
            .product(name: "Temporal", package: "swift-temporal-sdk"),
            .product(name: "<Service>Protos", package: "<project>-protos"),
        ]
    ),
],
swiftLanguageModes: [.v6]
```

Include a direct product dependency in every target that imports its module. The executable — not the feature target — needs `GRPCNIOTransportHTTP2` because it constructs the transport. Remove any dependency a target does not import.

Do not expose internal library products by default. The executables are the package products — `<service>`, and `<service>-worker` when the service uses Temporal:

```swift
products: [
    .executable(name: "<service>", targets: ["<Service>"]),
    .executable(name: "<service>-worker", targets: ["<Service>Worker"]), // only with Temporal
]
```

Internal targets communicate through `package` declarations. The worker is a separate executable rather than a subcommand so that it links only what it uses: the manifest edge, not discipline, is what keeps a database driver out of it. Its `+ConfigReader` extensions are copies, not a shared target — a shared configuration target would carry the Postgres extension and its driver back into the worker.

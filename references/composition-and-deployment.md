# Composition roots and deployment

## Contents

- Command tree
- Environment configuration
- Serve composition root
- Temporal worker composition root
- Migration composition root
- Container build
- Docker Compose topology

## Command tree

Name the executable target after the service, not `<Service>Server`. Make the root command default to serving and expose database migration as a nested subcommand:

```text
catalog                 # defaults to serve
catalog serve
catalog worker          # only when the service uses Temporal
catalog database migrate
```

```swift
@main
struct Catalog: AsyncParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "catalog",
        abstract: "Catalog Service",
        subcommands: [
            Serve.self,
            Worker.self,
            Database.self
        ],
        defaultSubcommand: Serve.self
    )
}
```

Put commands in `<Service>/<Command>/`, except the small root `Database.swift` at `<Service>/Database/Database.swift` with `Migrate` nested below it.

Register `Worker.self` as a root subcommand when the package uses Temporal. Keep `serve` as the default subcommand.

## Environment configuration

Create `PostgresClient.Configuration+ConfigReader.swift` in the executable's `Configuration` folder:

```swift
extension PostgresClient.Configuration {
    init(config: ConfigReader) throws {
        let host = try config.requiredString(forKey: "host")
        let port = try config.requiredInt(forKey: "port")
        let username = try config.requiredString(forKey: "user")
        let password = try config.requiredString(forKey: "password", isSecret: true)
        let database = try config.requiredString(forKey: "db")

        self.init(
            host: host,
            port: port,
            username: username,
            password: password,
            database: database,
            tls: .prefer(.clientDefault)
        )
    }
}
```

Start with `ConfigReader(provider: EnvironmentVariablesProvider())`, then scope to `postgres` or `grpc.server`. Swift Configuration transforms camel-case scoped keys into variables such as:

```dotenv
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=catalog
POSTGRES_PASSWORD=catalog
POSTGRES_DB=catalog
GRPC_SERVER_HOST=0.0.0.0
GRPC_SERVER_PORT=50051
TEMPORAL_HOST=localhost
TEMPORAL_PORT=7233
TEMPORAL_NAMESPACE=default
TEMPORAL_TASK_QUEUE=catalog
LOG_LEVEL=info
```

Require infrastructure values and secrets. Defaults are acceptable for the listen address, listen port, and log level. Put safe development examples in `.env.example`; ignore real `.env` files.

## Serve composition root

Use these section comments and this order:

```swift
func run() async throws {
    // MARK: - Configuration
    // MARK: - Logging
    // MARK: - Infrastructure
    // MARK: - Composition
    // MARK: - gRPC
    // MARK: - Lifecycle
}
```

Under Infrastructure, construct one `PostgresClient`. Under Composition, construct `PostgresDatabase<Postgres<Service>Context>`, use cases, and gRPC service implementations. Under gRPC, construct one server:

```swift
let serverConfig = config.scoped(to: "grpc.server")
let host = serverConfig.string(forKey: "host", default: "0.0.0.0")
let port = serverConfig.int(forKey: "port", default: 50051)
let server = GRPCServer(
    transport: .http2NIOPosix(
        address: .ipv4(host: host, port: port),
        transportSecurity: .plaintext
    ),
    services: [itemService]
)
```

Own all long-lived services with ServiceLifecycle:

```swift
let serviceGroup = ServiceGroup(
    services: [
        postgresClient,
        server
    ],
    gracefulShutdownSignals: [.sigint, .sigterm],
    logger: logger
)
```

Use plaintext on the private Docker/platform network when ingress terminates TLS. Add end-to-end or mutual TLS only when the deployment trust model requires it.

When the service uses Temporal, construct one long-lived `TemporalClient` in `serve`, inject a `<Feature>WorkflowClient` adapter into Core use cases, and include the client in `ServiceGroup`. Do not run workflow definitions or Activity implementations in the gRPC server process.

## Temporal worker composition root

Run Temporal execution from the executable's `Worker.self` subcommand. Follow the same Configuration, Logging, Infrastructure, Composition, and Lifecycle section order as `serve`.

Construct the databases and remote clients required by Activities once, compose the Core Activity service, then create one `TemporalWorker`:

```swift
let temporalWorker = try TemporalWorker(
    configuration: .init(
        namespace: temporalConfig.string(forKey: "namespace", default: "default"),
        taskQueue: temporalConfig.string(forKey: "taskQueue", default: "catalog"),
        instrumentation: .init(serverHostname: temporalHost)
    ),
    target: .dns(
        host: temporalHost,
        port: temporalConfig.int(forKey: "port", default: 7233)
    ),
    transportSecurity: .plaintext,
    activityContainers: ReservationActivities(service: activityService),
    workflows: [ReservationWorkflow.self],
    logger: logger
)
```

Add the worker and every long-lived dependency used by Activities to one `ServiceGroup`. Do not add a five-second registration scanner or another periodic database-to-Temporal reconciliation service. Temporal owns durable workflow execution.

## Migration composition root

The migration command uses the same configuration extension and logging system. Start the `PostgresClient` in a throwing task group, add migrations explicitly in order, apply them, then cancel the group:

```swift
try await withThrowingTaskGroup { group in
    group.addTask {
        await client.run()
    }

    let migrations = DatabaseMigrations()
    await migrations.add(CreateItemsTable())

    try await migrations.apply(client: client, logger: logger, dryRun: false)
    group.cancelAll()
}
```

## Container build

Use this Makefile shape:

```make
TAG ?= latest

build:
	swift package --swift-sdk aarch64-swift-linux-musl \
		--configuration release \
		--allow-network-connections all build-container-image \
		--product catalog \
		--repository ghcr.io/<organization>/<organization>-catalog \
		--tag $(TAG)

.PHONY: build
```

Replace only service/product/repository names.

## Docker Compose topology

For each extracted service create:

1. `<service>-postgres` using `postgres:18`, dedicated credentials, health check, and a dedicated volume mounted at `/var/lib/postgresql`.
2. `<service>-migrate` using the service image and `command: ["database", "migrate"]`, gated on healthy Postgres.
3. `<service>` using `command: ["serve"]`, gated on successful migration and exposing port `50051` only to the internal network.
4. `<service>-worker` using the same image with `command: ["worker"]` when Temporal is enabled; give it the same database, dependency, and Temporal configuration required by its Activities.
5. Consumer environment values `GRPC_<SERVICE>_HOST=<service>` and `GRPC_<SERVICE>_PORT=50051`, with startup gated on the producer.

PostgreSQL 18 changed its image volume layout. For the current convention, mount the volume at `/var/lib/postgresql`; do not set custom `PGDATA` for the extracted service.

Use dedicated variables such as `CATALOG_POSTGRES_USER`, `CATALOG_POSTGRES_PASSWORD`, and `CATALOG_POSTGRES_DB` in a larger deployment compose file, then map them to the service's expected `POSTGRES_*` variables. Never reuse the monolith database credentials merely because both services run in one Compose project.

Compose dependency conditions help startup ordering; they do not replace runtime recovery. The application must tolerate dependency restarts according to grpc-swift and Postgres client behavior.

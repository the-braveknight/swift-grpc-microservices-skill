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

A worker has no inbound request to forward, so it speaks as itself (see *Presenting a process's own identity* in identity-and-access.md). Under Infrastructure, construct a client to the issuer with no interceptor — the exchange authenticates by the secret in the request — and a `ServiceIdentitySession` over `ServiceIdentity`'s `IssueServiceToken` adapter; put `ServiceIdentityInterceptor(session:)` on every client the worker calls as itself. Under Lifecycle, race one exchange against the group so a wrong secret or an unreachable issuer fails the worker at startup naming the problem:

```swift
try await withThrowingTaskGroup { group in
    group.addTask { try await serviceGroup.run() }

    let token = try await serviceIdentitySession.refresh()
    logger.info("Authenticated as a service", metadata: ["expiration": "\(token.expirationDate)"])

    try await group.waitForAll()
}
```

`GRPCClient` tolerates an RPC that races its `run()`, so the exchange may start before the group has fully started its clients. Read the credential through a `ServiceIdentityCredentials+ConfigReader.swift` beside the other configuration extensions: `SERVICE_CREDENTIAL_ID` and `SERVICE_CREDENTIAL_SECRET_PATH`, the file's trailing newline trimmed.

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

When the service confines callers with row-level security, the migration command also reads `POSTGRES_APP_ROLE` (a default is fine) and `POSTGRES_APP_PASSWORD` (required) and registers the role migration after the table migrations and before the policies. `serve` then connects as that role: its `POSTGRES_USER` and `POSTGRES_PASSWORD` are the application role's, not the owner's the migration used.

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
3. `<service>` using `command: ["serve"]`, gated on successful migration and listening on `50051` inside the network.
4. `<service>-worker` using the same image with `command: ["worker"]` when Temporal is enabled; give it the same database, dependency, and Temporal configuration required by its Activities.
5. Consumer environment values `GRPC_<SERVICE>_HOST=<service>` and `GRPC_<SERVICE>_PORT=50051`, with startup gated on the producer.

PostgreSQL 18 changed its image volume layout. For the current convention, mount the volume at `/var/lib/postgresql`; do not set custom `PGDATA` for the extracted service.

Use dedicated variables such as `CATALOG_POSTGRES_USER`, `CATALOG_POSTGRES_PASSWORD`, and `CATALOG_POSTGRES_DB` in a larger deployment compose file, then map them to the service's expected `POSTGRES_*` variables. Never reuse the monolith database credentials merely because both services run in one Compose project.

Compose dependency conditions help startup ordering; they do not replace runtime recovery. The application must tolerate dependency restarts according to grpc-swift and Postgres client behavior.

Publish only what something outside the stack calls. Give Postgres and the Temporal server no `ports:` at all — services reach them by name over the compose network, and an unpublished port cannot collide with whatever already holds `5432` or `7233` on the host. Make each published host port a variable with a default so a conflict is settled in `.env` rather than by editing the file, and reach an internal service with `docker compose exec` instead of reopening a port.

A suite-level compose file runs published images and never builds them: `${REGISTRY:-...}/<image>:${IMAGE_TAG:-latest}` with `pull_policy`. Declare a required secret as `${VAR:?message}` so Compose refuses to start with an actionable error rather than a guessable default, and ship a `.env.example` naming every variable with the required ones left empty, copied to a git-ignored `.env`.

A service that connects as an application role overrides the credentials the shared anchor merges in — an explicit key in a mapping wins over `<<:` — while its migration job keeps the owner's and adds the role's:

```yaml
entitlements-migrate:
  environment:
    <<: *postgres-connection
    POSTGRES_APP_ROLE: ${ENTITLEMENTS_APP_ROLE:-entitlements_app}
    POSTGRES_APP_PASSWORD: ${ENTITLEMENTS_APP_PASSWORD:-emberfilm}

entitlements:
  environment:
    <<: [*postgres-connection, *jwt-verification]
    POSTGRES_USER: ${ENTITLEMENTS_APP_ROLE:-entitlements_app}
    POSTGRES_PASSWORD: ${ENTITLEMENTS_APP_PASSWORD:-emberfilm}
```

A process's service credential is a mounted secret like a key, and it is the one secret that cannot exist before the stack has run: it is issued against the issuer's migrated database. Compose refuses to start a service whose secret file is missing, so the first start is staged, and the `.env.example` says so:

```sh
docker compose up -d postgres postgres-init authentication-migrate authentication
docker compose run --rm authentication service-credentials create --id authentication-worker \
    > secrets/authentication-worker.secret
docker compose up -d
```

`docker compose config` validates a file whose secret source is missing; only `up` refuses. Every service that verifies tokens must be the build that decodes the process's role before that process is started, or its every call is refused.

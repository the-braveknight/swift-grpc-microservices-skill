# Composition roots

## Contents

- Command tree
- Environment configuration
- Serve composition root
- Temporal worker composition root
- Migration composition root

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

Require infrastructure values and secrets. Defaults are acceptable for the listen address, listen port, and log level. Where the values come from in a running environment is [environment.md](environment.md)'s subject.

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

Under Logging, bootstrap the logging system **inline** — never hide it behind a shared helper or module. Build one `LokiLogProcessor`, then `LoggingSystem.bootstrap` a `MultiplexLogHandler` of `StreamLogHandler.standardOutput` and a `LokiLogHandler`, so every line reaches both the container's stdout and Loki. Pass the service name to the Loki handler's `service:` so its lines carry a `service` label. Each service keeps its own `LokiLogProcessorConfiguration+ConfigReader.swift` beside its other configuration extensions (`config.scoped(to: "loki")`, `LOKI_URL`, defaulting to `http://localhost:3100`) — a small extension duplicated per service, not a shared observability package. Do not extract a shared logging module; the duplication is cheaper than the coupling.

```swift
// MARK: - Logging
let lokiProcessor = LokiLogProcessor(
    configuration: LokiLogProcessorConfiguration(config: config.scoped(to: "loki"))
)
let logLevel = config.string(forKey: "logLevel", default: Logger.Level.info)
LoggingSystem.bootstrap { label in
    var handler = MultiplexLogHandler([
        StreamLogHandler.standardOutput(label: label),
        LokiLogHandler(label: label, service: "catalog", processor: lokiProcessor),
    ])
    handler.logLevel = logLevel
    return handler
}
let logger = Logger(label: "catalog")
```

Under Infrastructure, construct one `PostgresClient` and scope the transport-security reader:

```swift
let tlsConfig = config.scoped(to: "tls")
```

Under Composition, construct `PostgresDatabase<Postgres<Service>Context>`, use cases, and gRPC service implementations. Under gRPC, construct one server:

```swift
let serverConfig = config.scoped(to: "grpc.server")
let host = serverConfig.string(forKey: "host", default: "0.0.0.0")
let port = serverConfig.int(forKey: "port", default: 50051)
let server = GRPCServer(
    transport: .http2NIOPosix(
        address: .ipv4(host: host, port: port),
        transportSecurity: try .mTLS(config: tlsConfig)
    ),
    services: [itemService]
)
```

Own all long-lived services with ServiceLifecycle:

```swift
let serviceGroup = ServiceGroup(
    services: [
        lokiProcessor,
        postgresClient,
        server
    ],
    gracefulShutdownSignals: [.sigint, .sigterm],
    logger: logger
)
```

Every connection is mutually authenticated, in both directions, with the one leaf certificate the process was issued (see *Transport security* in environment.md). The factories live in the executable's `Configuration` folder as `TransportSecurity+ConfigReader.swift`, one per direction, on grpc-swift's own types — so a call site reads exactly like the library's `.plaintext` did, and there is no struct to carry two values and no mode to switch:

```swift
extension HTTP2ServerTransport.Posix.TransportSecurity {
    /// A server cannot know a client's hostname, so it checks only that the client's certificate
    /// chains to the CA — grpc's default for mTLS.
    static func mTLS(config: ConfigReader) throws -> Self {
        let certificateChain: [TLSConfig.CertificateSource] = [
            .file(path: try config.requiredString(forKey: "certificatePath"), format: .pem)
        ]
        let privateKey: TLSConfig.PrivateKeySource = .file(
            path: try config.requiredString(forKey: "privateKeyPath"),
            format: .pem
        )
        let trustRoots: TLSConfig.TrustRootsSource = .certificates([
            .file(path: try config.requiredString(forKey: "trustRootsPath"), format: .pem)
        ])
        return .mTLS(certificateChain: certificateChain, privateKey: privateKey) { tls in
            tls.trustRoots = trustRoots
        }
    }
}

extension HTTP2ClientTransport.Posix.TransportSecurity {
    /// A client knows exactly whom it dialled, so it checks the name as well as the chain.
    static func mTLS(config: ConfigReader) throws -> Self {
        // the same three sources
        return .mTLS(certificateChain: certificateChain, privateKey: privateKey) { tls in
            tls.trustRoots = trustRoots
            tls.serverCertificateVerification = .fullVerification
        }
    }
}
```

The reader is scoped to `tls`, so the operator sets `TLS_CERTIFICATE_PATH`, `TLS_PRIVATE_KEY_PATH` and `TLS_TRUST_ROOTS_PATH` — paths, like the signing key, because that is the form NIOSSL takes credentials in and it keeps a private key out of the environment. A missing one fails at startup naming the key. Every `GRPCClient`, and the `TemporalClient` and `TemporalWorker` when the Temporal server runs inside the stack, take the client factory; the `GRPCServer` takes the server one. When the Temporal server is Temporal Cloud, the client factory is wrong on both counts — its trust roots are the internal CA, and Cloud's frontend chains to a public one — so the Temporal client and worker take TLS with the system trust roots and authenticate with an API key instead:

```swift
let temporalClient = try TemporalClient(
    target: .dns(host: temporalHost, port: 7233),        // <namespace>.<account>.tmprl.cloud
    transportSecurity: .tls { tls in tls.trustRoots = .systemDefault },
    configuration: .init(
        instrumentation: .init(serverHostname: temporalHost),
        namespace: temporalNamespace,
        // the API key, from the `temporal` scope, as the SDK's authorization option or an
        // interceptor adding `Authorization: Bearer` — verify which the pinned SDK exposes
    ),
    logger: logger
)
```

Keep the two concerns in two scopes: `tls` is who the process is inside the stack, `temporal` is where the workflow engine is and how it is reached. A namespace on Temporal Cloud can also be configured for certificate authentication, which is mTLS again but against a CA registered with the namespace; prefer the API key, which rotates from the Cloud console and binds no vendor setting to the stack's CA. Do not share the file through the identity package: the shape is eight lines a service owns, and a shared package change costs a tag and a bump in every consumer.

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
    transportSecurity: try .mTLS(config: tlsConfig),
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

The migration command uses the same configuration extension. It is short-lived and owns no `ServiceGroup`, so it does not ship to Loki — bootstrap `StreamLogHandler.standardOutput` alone. Start the `PostgresClient` in a throwing task group, add migrations explicitly in order, apply them, then cancel the group:

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

The migration command also reads `POSTGRES_SERVICE_ROLE` (a default of `<service>_service` is fine) and `POSTGRES_SERVICE_PASSWORD` (required) and registers `CreateServiceRole` first. Every other command connects as that role: its `POSTGRES_USER` and `POSTGRES_PASSWORD` are the service role's, not the owner's the migration used.

How a running environment supplies these values — images, secrets, ports, orchestration — is in [environment.md](environment.md) and nowhere else.

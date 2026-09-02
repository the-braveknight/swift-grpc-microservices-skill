# Composition roots

## Contents

- Command tree
- Environment configuration
- Serve composition root
- Transport security factories
- Temporal worker composition root
- Migration composition root
- Operator commands

## Command tree

Name the executable target after the service, not `<Service>Server`. The root command defaults to serving, and there is no migrate subcommand: migrations are a `serve` flag, applied in-process before the server binds (see *Migrations at boot* below).

```text
catalog                          # defaults to serve
catalog serve
catalog serve --migrate-database # apply pending migrations, then serve
catalog-worker                   # only with Temporal — a second executable, one command
catalog service-credentials create --id <name>   # only on the authenticating service
```

```swift
@main
struct Catalog: AsyncParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "catalog",
        abstract: "Catalog Service",
        subcommands: [
            Serve.self
        ],
        defaultSubcommand: Serve.self
    )
}
```

Put commands in `<Service>/<Command>/`; the migration list lives at `<Service>/Database/Migrations.swift`.

## Environment configuration

Start with `ConfigReader(provider: EnvironmentVariablesProvider())`, then scope by concern. Swift Configuration transforms camel-case scoped keys into variables:

```dotenv
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=catalog_service
POSTGRES_PASSWORD=…
POSTGRES_DB=catalog
GRPC_SERVER_HOST=0.0.0.0
GRPC_SERVER_PORT=50051
GRPC_ACCOUNTS_HOST=accounts          # one scope per upstream, required
GRPC_ACCOUNTS_PORT=50051
TLS_CERTIFICATE_PATH=/run/tls/cert.pem
TLS_PRIVATE_KEY_PATH=/run/tls/key.pem
TLS_TRUST_ROOTS_PATH=/run/tls/ca.pem
JWT_PUBLIC_KEY_PATH=/run/secrets/jwt-public
TEMPORAL_HOST=temporal
TEMPORAL_PORT=7233
TEMPORAL_NAMESPACE=default
TEMPORAL_TASK_QUEUE=catalog
LOKI_URL=http://loki:3100
LOG_LEVEL=info
```

Give each configuration an `init(config:)` extension in the executable's `Configuration` folder — `PostgresClient.Configuration+ConfigReader.swift`, `TransportSecurity+ConfigReader.swift`, `IdentityVerifier.Configuration+ConfigReader.swift`, and so on. A small extension duplicated per service, not a shared package.

```swift
extension PostgresClient.Configuration {
    init(config: ConfigReader) throws {
        self.init(
            host: try config.requiredString(forKey: "host"),
            port: try config.requiredInt(forKey: "port"),
            username: try config.requiredString(forKey: "user"),
            password: try config.requiredString(forKey: "password", isSecret: true),
            database: try config.requiredString(forKey: "db"),
            tls: .prefer(.clientDefault)
        )
    }
}
```

Require infrastructure values and secrets. Defaults are acceptable for the listen address, listen port, and log level. Require an upstream's host and port rather than defaulting them: a process that quietly dials `localhost` in a container reports a misconfiguration as a connection failure minutes later, at the first request, instead of at startup.

Key material — signing keys, certificates, service credentials — is configured as a path and the file is opened here, in the composition root, never in a library. A path is the form NIOSSL and grpc-swift already take credentials in, it keeps a private key out of the environment, and it fails at startup naming the path. The rationale is in [identity-and-access.md](identity-and-access.md), *Key material in configuration*.

Where the values come from in a running environment is [environment.md](environment.md)'s subject.

## Serve composition root

Use these section comments in this order:

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

**Logging.** Bootstrap the logging system inline — never behind a shared helper or module. Build one in-process log shipper, then `LoggingSystem.bootstrap` a `MultiplexLogHandler` of `StreamLogHandler.standardOutput` and the shipper's handler, so every line reaches both the container's stdout and the aggregator. Pass the service name as the handler's service label. The default aggregator is Grafana Loki through `swift-log-loki`; substituting another in-process shipper changes only this block.

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

**Infrastructure.** Construct one `PostgresClient`, the identity verifier, one `GRPCClient` per upstream, and — with Temporal — one `TemporalClient`. Scope the transport-security reader once:

```swift
let tlsConfig = config.scoped(to: "tls")
```

**Composition.** Construct `PostgresDatabase<Postgres<Service>Context>`, policies, use cases (passing `logger`), workflow-client adapters, and gRPC service implementations.

**gRPC.** Construct one server, with the identifying interceptor applied to everything except the session-issuing RPCs:

```swift
let serverConfig = config.scoped(to: "grpc.server")
let server = GRPCServer(
    transport: .http2NIOPosix(
        address: .ipv4(
            host: serverConfig.string(forKey: "host", default: "0.0.0.0"),
            port: serverConfig.int(forKey: "port", default: 50051)
        ),
        transportSecurity: try .mTLS(config: tlsConfig)
    ),
    services: [itemService],
    interceptorPipeline: [
        .apply(
            IdentityServerInterceptor(verifier: identityVerifier),
            to: .allExcluding(services: [], methods: ItemService.publicMethods)
        )
    ]
)
```

**Lifecycle.** Own every long-lived thing with ServiceLifecycle:

```swift
let serviceGroup = ServiceGroup(
    services: [lokiProcessor, postgresClient, accountsClient, server],
    gracefulShutdownSignals: [.sigint, .sigterm],
    logger: logger
)
try await serviceGroup.run()
```

When the service uses Temporal, construct one long-lived `TemporalClient` in `serve`, inject a `<Feature>WorkflowClient` adapter into Core use cases, and include the client in `ServiceGroup`. Do not run workflow definitions or Activity implementations in the gRPC server process.

## Transport security factories

Every connection is mutually authenticated, in both directions, with the one leaf certificate the process was issued (see *Transport security* in [environment.md](environment.md)). The factories live in the executable's `Configuration` folder as `TransportSecurity+ConfigReader.swift`, one per direction, on grpc-swift's own types — so a call site reads exactly like the library's `.plaintext` did, and there is no struct to carry two values and no mode to switch:

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

The reader is scoped to `tls`, so the operator sets `TLS_CERTIFICATE_PATH`, `TLS_PRIVATE_KEY_PATH`, and `TLS_TRUST_ROOTS_PATH`. A missing one fails at startup naming the key. Every `GRPCClient`, and the `TemporalClient` and `TemporalWorker` when the Temporal server runs inside the stack, take the client factory; the `GRPCServer` takes the server one. Do not share the file through the identity package: the shape is eight lines a service owns.

**A managed workflow engine or any external endpoint is outside the stack.** The client factory is wrong on both counts for it — its trust roots are the internal CA, and the external frontend chains to a public one. Such a client uses TLS with the system trust roots and the provider's own credential, configured in that provider's scope beside its address:

```swift
let temporalClient = try TemporalClient(
    target: .dns(host: temporalHost, port: 7233),        // the provider's endpoint
    transportSecurity: .tls { tls in tls.trustRoots = .systemDefault },
    configuration: .init(
        instrumentation: .init(serverHostname: temporalHost),
        namespace: temporalNamespace,
        // the API key, from the `temporal` scope, through whichever option the pinned SDK exposes
    ),
    logger: logger
)
```

Keep the two concerns in two scopes: `tls` is who the process is inside the stack; `temporal` is where the workflow engine is and how it is reached. Prefer an API key over registering a CA with the provider: it rotates from the provider's console and binds no vendor setting to the stack's CA.

## Temporal worker composition root

The worker is its own executable target, `<Service>Worker`, with product `<service>-worker`, in the same package. Its `@main` is a single `AsyncParsableCommand`, `Worker` in `Worker.swift`, named `<service>-worker` on the command line, whose `run()` is the composition root, in the same section order as `serve`. It has its own `Configuration/` folder holding copies of the `+ConfigReader` extensions it reads — transport security, the log shipper, the service credential, a provider client — and none for Postgres, because it links none:

```swift
@main
struct Worker: AsyncParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "catalog-worker",
        abstract: "Catalog Worker"
    )

    func run() async throws {
        // MARK: - Configuration
        // MARK: - Logging
        // MARK: - Infrastructure
        // MARK: - Composition
        // MARK: - Worker
        // MARK: - Lifecycle
    }
}
```

It builds to its own image, `<organization>-<service>-worker`, from the same `Makefile` and the same `.build`; deploy it at the same tag as the service, because it speaks the service's own worker-facing contract.

The worker owns no database (see *Worker composition* in [temporal-workflows.md](temporal-workflows.md)). Under Infrastructure, construct the long-lived gRPC and provider clients its Activities call. Under Composition, build the concrete Core Activity service over those clients. Then create one `TemporalWorker`:

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

Add the worker and every long-lived dependency used by Activities to one `ServiceGroup`. Do not add a periodic database-to-Temporal reconciliation service. Temporal owns durable workflow execution.

A worker has no inbound request to forward, so it speaks as itself (see *Presenting a process's own identity* in [identity-and-access.md](identity-and-access.md)). Under Infrastructure, construct a client to the issuer with no interceptor — the exchange authenticates by the secret in the request — and a `ServiceIdentitySession` over the `ServiceIdentity` product's `IssueServiceToken` adapter; put `ServiceIdentityInterceptor(session:)` on every client the worker calls as itself. Under Lifecycle, race one exchange against the group so a wrong secret or an unreachable issuer fails the worker at startup naming the problem:

```swift
try await withThrowingTaskGroup { group in
    group.addTask { try await serviceGroup.run() }

    let token = try await serviceIdentitySession.refresh()
    logger.info("Authenticated as a service", metadata: ["expiration": "\(token.expirationDate)"])

    try await group.waitForAll()
}
```

`GRPCClient` tolerates an RPC that races its `run()`, so the exchange may start before the group has fully started its clients. Read the credential through `ServiceIdentityCredentials+ConfigReader.swift`: `SERVICE_CREDENTIAL_ID` and `SERVICE_CREDENTIAL_SECRET_PATH`, the file's trailing newline trimmed. The same shape applies to any process that calls as itself — a webhook-handling `serve`, a scheduled job.

## Migrations at boot

`serve --migrate-database` applies migrations after logging bootstrap and before any long-lived infrastructure exists. Each service has its own Postgres instance, so the postgres scope is read once into one executable-local configuration that derives both connections:

```swift
struct PostgresConfiguration: Sendable {
    let host: String; let port: Int; let database: String
    let user: String; let password: String               // the instance's own pair — the owner
    let serviceUser: String; let servicePassword: String // the confined role serve enters as

    init(config: ConfigReader) throws { /* the one place the scope is read */ }

    var owner: PostgresClient.Configuration { /* username: user */ }
    var service: PostgresClient.Configuration { /* username: serviceUser */ }
}
```

The owner client lives exactly as long as the apply, through a scoped helper that starts it in a task group and cancels it when the operation returns:

```swift
if migrateDatabase {
    try await PostgresClient.withClient(configuration: postgres.owner, logger: logger) { client in
        try await Migrations(client: client, configuration: postgres, logger: logger).run()
    }
}
```

`Migrations.run()` adds the list explicitly in order — `CreateServiceRole` first, from the service pair — and applies it. The long-lived client the `ServiceGroup` owns is built from `postgres.service` and never holds owner credentials; the owner pair does sit in the serving container's environment, which is the accepted price of migrating in-process — the *process* that serves never connects with it.

## Operator commands

An operation that must never be reachable over the network — issuing a service credential, rotating a key — is a subcommand, run by an operator against the service's own database. It follows the boot-migration shape: short-lived, stdout logging, a scoped client around the work. Print exactly the value the operator needs on standard output and nothing else there, so a redirect captures it. See *Service credentials and issuance* in [identity-and-access.md](identity-and-access.md).

# API gateway

The one HTTP surface in front of a gRPC system. It terminates HTTP, identifies the caller,
translates each request into an RPC, and translates the reply back.

## Contents

- Naming and target graph
- Exact source tree
- The OpenAPI document is the contract
- Request contexts
- Router tiers
- Route design
- Controllers
- Schema conversion
- Error translation
- Serve composition root
- Key material as a path
- One surface or two
- Deployment

## Naming and target graph

Name the package `<organization>-api`, not `-gateway`. The naming grammar already refuses
`gateway` as a type name, and the thing that actually gateways — routing, TLS termination,
rate limiting — is the ingress in front of this process. What this package holds is
hand-written controllers mapping one contract onto another, which is an edge service.

```text
<organization>-api
├── API ────────→ generated OpenAPI types, contexts, middleware, controllers, conversions
└── <Project> ──→ API   # command tree, configuration, clients, lifecycle
```

Two targets, not four. A gateway owns no entities, no repositories, and no database, so
`<Service>Core` and `<Service>Postgres` would be empty. Do not add an `APICore` to mirror the
service shape: the only candidates for it are the request contexts and the problem types, and
both are transport concerns that belong beside the controllers that use them.

The split that does matter is the same one every service makes — a library holding the surface,
and an executable holding the composition root. Keep `@main`, `ConfigReader`, and client
construction out of the library.

| Target | Owns | Must not own |
| --- | --- | --- |
| `API` | Generated OpenAPI types, request contexts, error middleware and problem types, controllers, protobuf conversions, route registration | Environment reading, client construction, logging setup, `@main` |
| `<Project>` | ArgumentParser command tree, configuration, client and verifier construction, router assembly, lifecycle | Route handlers, schema conversion, business rules |

## Exact source tree

```text
Sources/
├── API/
│   ├── openapi.yaml
│   ├── openapi-generator-config.yaml
│   ├── <Project>API.swift
│   ├── Contexts/
│   │   ├── IdentityRequestContext.swift
│   │   └── AdminRequestContext.swift
│   ├── Controllers/
│   │   ├── AuthenticationController.swift
│   │   ├── ItemController.swift
│   │   └── ProfileController.swift
│   ├── Middlewares/
│   │   └── ErrorMiddleware/
│   │       ├── ErrorMiddleware.swift
│   │       └── Problem/
│   │           ├── Problem.swift
│   │           ├── HTTPProblemResponse.swift
│   │           └── Conformances/
│   │               ├── HTTPError+HTTPProblemResponse.swift
│   │               └── RPCError+HTTPProblemResponse.swift
│   └── Schemas/
│       ├── Requests/
│       │   └── CreateItemRequest+RPC.swift
│       └── Responses/
│           └── ItemResponse+RPC.swift
└── <Project>/
    ├── <Project>.swift
    ├── Serve/
    │   └── Serve.swift
    └── Configuration/
        └── IdentityVerifier.Configuration+ConfigReader.swift
```

## The OpenAPI document is the contract

Keep `openapi.yaml` in the target directory beside `openapi-generator-config.yaml`. That is
where the generator plugin looks. Do not keep it elsewhere and reach for it with
`resources: [.copy("../../Docs/openapi.yaml")]`: the relative path escapes the target
directory, which SwiftPM warns about, and the copy exists only to satisfy a lookup the
convention already performs.

Generate types only:

```yaml
generate:
  - types
accessModifier: package
```

The gateway registers its own routes on Hummingbird, so generated server stubs would be a
second routing mechanism maintained for the same endpoints. Use `package`, not `public` — the
types cross one target boundary inside one package.

Unimplemented paths in the document cost nothing while types are all that is generated, so the
document can lead the implementation and serve as the porting checklist.

## Request contexts

Chain three, each narrowing what the handler is allowed to assume:

```swift
package struct IdentityRequestContext: ChildRequestContext, AuthRequestContext {
    package typealias ParentContext = BasicRequestContext

    package var coreContext: CoreRequestContextStorage
    package var identity: Identity?

    package init(context: BasicRequestContext) throws {
        self.coreContext = context.coreContext
        self.identity = nil
    }
}
```

Carry `coreContext` across. Do not rebuild it with `.init(source: context)`, which is what
Hummingbird's own `ChildRequestContext` documentation shows: `CoreRequestContextStorage`'s
`init(source:)` starts `parameters` and `endpointPath` empty, and path parameters are extracted
before the child is constructed. Rebuilding silently discards them, so a route declared with
`:id` matches, runs, and then cannot find its own parameter.

The admin context holds a non-optional identity, because reaching a handler through it is the
proof the check ran:

```swift
package struct AdminRequestContext: ChildRequestContext {
    package typealias ParentContext = IdentityRequestContext

    package var coreContext: CoreRequestContextStorage
    package let identity: Identity

    package init(context: IdentityRequestContext) throws {
        guard let identity = context.identity else {
            throw HTTPError(.unauthorized, message: "Authentication is required.")
        }
        guard identity.role == .admin else {
            throw HTTPError(.forbidden, message: "This operation requires an administrator.")
        }
        self.coreContext = context.coreContext
        self.identity = identity
    }
}
```

Distinguish the two failures for the same reason gRPC handlers do. Absent is `401`, because
presenting a token could change the answer; insufficient is `403`, because it could not.

The surface is for people. Every route that acts on "the caller's own" account sends
`identity.subject` upstream as a user id, and a process's token — `role: service` — carries a
credential name there instead. Give `IdentityRequestContext` a `requireUser()` and use it, not
`requireIdentity()`, on those routes:

```swift
package func requireUser() throws -> Identity {
    let identity = try requireIdentity()
    guard identity.role != .service else {
        throw HTTPError(.forbidden, message: "This operation requires a user.")
    }
    return identity
}
```

`403`, because the token is valid and is simply not the kind the route takes; and here rather than
upstream, because a users service would otherwise report the name as a malformed id — the wrong
error for the right refusal. The administrative conversion already refuses anything but `admin`,
so a process is turned away on every route without a per-handler test.

## Router tiers

Register routes in three tiers, and let the tier — not a path prefix or a path exception —
decide what runs:

```swift
let router = Router(context: BasicRequestContext.self)
router.add(middleware: CORSMiddleware(...))
router.add(middleware: ErrorMiddleware())
router.add(middleware: LogRequestsMiddleware(.info))

let v1 = router.group("v1")

// Tier 1 — no caller. Session-issuing routes only.
authenticationController.addPublicRoutes(to: v1.group("auth"))

// Tier 2 — a caller if there is one; anonymous requests still arrive.
let identified = v1.group(context: IdentityRequestContext.self)
    .add(middleware: IdentityAuthenticator(verifier: identityVerifier))

// Tier 3 — a caller is required.
let authenticated = identified.add(middleware: IsAuthenticatedMiddleware())
```

Tier 1 exists for the same reason session-issuing RPCs are excluded from the gRPC identifying
interceptor: it is that rule, one transport over. Token refresh sends the refresh token in the
`Authorization` header, and a refresh token is a database row rather than a signed one. An
authenticating middleware applied to it verifies that value as a claim payload, fails, and
returns `401` before the handler is reached — so the route cannot succeed at any point, for any
client. Grouping by tier prevents that structurally. A path exception inside the middleware does
not: it restates the rule in a second place that drifts.

Tier 2 is where a route that reads differently for a known caller belongs — a catalog that is
public but richer once logged in. Do not collapse tiers 2 and 3; identifying and requiring are
separate decisions here exactly as they are on gRPC.

## Route design

Authorization is a property of the method-and-resource pair, not of a path prefix. A parallel
`/admin/...` tree makes `/admin/users` and `/users` two resources when they are one resource
with two audiences, and it splits the document into two shapes for the same thing.

Flatten it, and let three mechanisms carry the distinction:

- **Method.** Reads open, writes admin: `GET /packages` alongside `POST /packages`.
- **Collection versus item.** `GET /users` is inherently an administrative capability where
  `GET /profile` is not.
- **Sub-resources.** `/users/{id}/entitlements` for an administrator, a caller-scoped route for
  everyone else.

Express the administrative half as a context group at the same path, never a path group:

```swift
package func addAuthenticatedRoutes(to group: RouterGroup<IdentityRequestContext>) {
    group
        .group(context: AdminRequestContext.self)
        .get(use: listUsers)
        .get(":id", use: getUser)
}
```

The conversion throws, so an administrative handler is unreachable without the check having
run, and the guarantee is in the type rather than in the order middleware happened to be added.

Two limits, both worth respecting rather than forcing through:

- Flattening fails where two operations share a method and path but differ in scope — a
  caller-scoped list and an administrative list of everything. Keep those distinct with a query
  parameter or a separate path; collapsing them produces one operation with two meanings.
- Not every route is a resource. Sessions, provider webhooks, and checkout callbacks are
  workflows, and `POST /payments/<provider>/webhook` is the honest spelling. Do not restructure a
  working verb-shaped route to satisfy a taxonomy.

Where a check depends on a path parameter — this record if it is yours, any record if you are an
administrator — it cannot be a context. Write it in the handler, and give it one name rather
than open-coding it at each call site.

## Controllers

One `XController` per resource, holding generated client protocols rather than concrete clients
so a test can substitute them:

```swift
package struct ItemController: Sendable {
    private let catalogClient: Catalog_V1_ItemService.ClientProtocol
    private let inventoryClient: Inventory_V1_StockService.ClientProtocol

    package func addPublicRoutes(to group: RouterGroup<BasicRequestContext>) { ... }
    package func addAuthenticatedRoutes(to group: RouterGroup<IdentityRequestContext>) { ... }
}
```

Name the registration methods for the tier they mount into. A controller may reach more than one
service — an item's record is owned by catalog while its stock level is owned by inventory — and
that is the gateway working: which service answers a resource is its problem, not the client's.

Keep the path prefix in the composition root and the routes in the controller, so one file shows
the whole surface and each controller stays movable.

## Schema conversion

Put conversions in `Schemas/Requests/` and `Schemas/Responses/` as `X+RPC.swift`, matching the
producer-side `Protobuf/` convention.

Make a conversion throw rather than drop:

```swift
extension Components.Schemas.ItemResponse: ResponseCodable {
    init(_ item: Catalog_V1_Item) throws {
        guard let id = UUID(uuidString: item.id) else {
            throw HTTPError(.internalServerError, message: "The item record is malformed.")
        }
        self.init(id: id, name: item.name)
    }
}
```

A `compactMap` over a malformed field returns a shorter list and a `200`, so a client sees a
record that has disappeared rather than an error anybody investigates. A value the upstream
service should never have produced is a fault in that service and should say so.

## Error translation

Answer every failure as RFC 9457 problem details with `application/problem+json`, and map
`RPCError` by code rather than collapsing it:

| gRPC code | HTTP |
| --- | --- |
| `invalidArgument`, `failedPrecondition`, `outOfRange` | 400 |
| `unauthenticated` | 401 |
| `permissionDenied` | 403 |
| `notFound` | 404 |
| `alreadyExists`, `aborted` | 409 |
| `resourceExhausted` | 429 |
| `unimplemented` | 501 |
| `unavailable` | 503 |
| `deadlineExceeded` | 504 |

The producing service already classified the failure into a status the contract defines. Mapping
every RPC failure to `500` discards that work at the last hop and tells a client to retry what
cannot succeed. Reserve `500` for a failure the gateway itself could not classify, and log that
one with the underlying error.

## Serve composition root

Follow the service section order, substituting the transport the gateway actually serves:

```swift
func run() async throws {
    // MARK: - Configuration
    // MARK: - Logging
    // MARK: - Infrastructure
    // MARK: - Composition
    // MARK: - Router
    // MARK: - Hummingbird
    // MARK: - Lifecycle
}
```

Under Infrastructure, build the verifier and one client per upstream service. Require the host
and port rather than defaulting them — a gateway that quietly dials `localhost` in a container
reports a misconfiguration as a connection failure minutes later, at the first request, instead
of at startup.

```swift
let usersConfig = config.scoped(to: "grpc.users")
let usersClient = GRPCClient(
    transport: try .http2NIOPosix(
        target: .dns(
            host: try usersConfig.requiredString(forKey: "host"),
            port: try usersConfig.requiredInt(forKey: "port")
        ),
        transportSecurity: .plaintext
    ),
    interceptors: [IdentityClientInterceptor()]
)
```

Register the forwarding interceptor on the clients that call on behalf of a caller, and leave it
off the client the session-issuing routes use. Those routes run before any caller exists, so
there is no token to forward and nothing for the interceptor to do.

Own the applications and every client in one `ServiceGroup`:

```swift
let serviceGroup = ServiceGroup(
    services: [authenticationClient, usersClient, application],
    gracefulShutdownSignals: [.sigint, .sigterm],
    logger: logger
)
```

Read the listener from configuration with `ApplicationConfiguration(reader:)`, scoped to
`http.server`.

## Key material as a path

Configure the verifying key by path and read the document in the composition root:

```swift
extension IdentityVerifier.Configuration {
    init(config: ConfigReader) throws {
        let publicKeyPath = try config.requiredString(forKey: "publicKeyPath")
        let publicKey = try String(contentsOfFile: publicKeyPath, encoding: .utf8)
        try self.init(publicKey: EdDSA.PublicKey(pem: publicKey))
    }
}
```

A path is the form the surrounding libraries already take. `TLSConfig.CertificateSource.file(path:format:)`
and `PrivateKeySource.file(path:format:)` in grpc-swift, and `NIOSSLCertificate.fromPEMFile` in
NIOSSL, are given a path and open it themselves — so the mTLS credentials a gateway is most
likely to need next are configured exactly like the key it needs now, and none of that material
becomes a configuration value an access reporter can write to a log.

It also removes the reason the document was ever base64 encoded: a path survives an environment
variable intact where a PEM's newlines do not. This is the same rule the services behind the
gateway follow — see [identity-and-access.md](identity-and-access.md) — so one key is
configured one way across the system.

Mount the file — how is [environment.md](environment.md)'s subject — and point the variable at
it. Do not bundle a key as a SwiftPM resource: it is not a leak for a public key, but it bakes the
value into the image and makes rotation a rebuild.

## One surface or two

Default to one target, one document, and one application. Authorization by context conversion is
enough to keep administrative routes out of ordinary hands, and a second surface doubles the
generated type set: two documents produce two unrelated Swift types for every schema they share.

Split into two Hummingbird applications — two routers, two ports, one `ServiceGroup` — only when
the administrative routes must not be publicly routable at all. That buys something a role check
cannot: the routes are absent from the public router's tree, so no ordering mistake can expose
them, and the port is simply never published.

Before splitting, check whether the administrative client removes the need. A browser client
served by its own process can proxy same-origin to an unpublished gateway port, which gets the
isolation without splitting Swift targets, and leaves the gateway with no CORS configuration to
maintain for it.

## Deployment

The gateway publishes no host port: a tailnet sidecar owns its address and proxies to it at
localhost, and upstream gRPC services are reachable by name on the internal network and by
nothing outside it. The sidecar, its serve configuration, and what a cutover behind that address
entails are in [environment.md](environment.md).

Configure CORS only for browsers that reach the gateway directly. A client whose own server
proxies to it is same-origin and needs none.

The gateway needs no database, so it has no migration job and no ordering constraint beyond its
upstreams being reachable. Give it a health route in tier 1 so the platform can probe it without
a token.

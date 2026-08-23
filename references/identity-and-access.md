# Identity and access

How a caller is identified across services, who may mint a token, and how the credential travels.

## The shared identity package

Put the claim payload, the signer, the verifier, and one adapter per transport in a shared
`<project>-identity` package beside `<project>-protos`, depended on by tagged URL. Products:

- `Identity` — the payload, the signer, the verifier, and the task local. Depends on JWTKit only.
- `IdentityGRPC` — the server and client interceptors. Depends on `Identity` and `GRPCCore`.
- `IdentityHTTP` — the authenticating middleware. Depends on `Identity` and Hummingbird.

It is a separate package from `<project>-protos` because it is runtime behavior rather than a
contract, and separate products because a gRPC-only service must not link Hummingbird to verify a
token. Core may link `Identity`; it is a focused shared package, not an infrastructure SDK.

Because it is consumed by tag, a source edit here is invisible to every service until it is tagged
and each consumer's `Package.resolved` is updated. Verify a cross-repo change before tagging by
pointing a consumer at the working copy:

```sh
swift package edit <project>-identity --path ../<project>-identity   # resolution must succeed first
swift build
swift package unedit <project>-identity
```

Adding a case to a public enum here breaks every consumer that switches over it exhaustively. Plan
it as a breaking change across the services, even though a `from:` requirement will resolve the new
tag automatically and present it as a build failure rather than a resolution one.

## One issuer, asymmetric keys

Sign with EdDSA (Ed25519). The service that authenticates users holds the private key and is the
only service that can mint a token; every other service is configured with the public key, which
verifies a token but cannot produce one.

Never use a shared HMAC secret. A symmetric key makes every service that can verify a token also
able to forge one, which erases the distinction the architecture depends on. Key distribution, not
a target boundary, is what keeps a single issuer: a service holding only the public key cannot
build a signer even when the signer type is in scope.

## The claim payload

```swift
public struct Identity: JWTPayload {
    public let subject: String
    public let role: UserRole
    public let issuer: IssuerClaim
    public let issuedAt: IssuedAtClaim
    public let expiration: ExpirationClaim

    init(
        subject: String,
        role: UserRole,
        issuer: IssuerClaim,
        issuedAt: IssuedAtClaim,
        expiration: ExpirationClaim
    ) { … }

    public func verify(using algorithm: some JWTAlgorithm) throws {
        try expiration.verifyNotExpired()
    }

    private enum CodingKeys: String, CodingKey {
        case subject = "sub"
        case role
        case issuer = "iss"
        case issuedAt = "iat"
        case expiration = "exp"
    }
}

public enum UserRole: String, Codable, Sendable {
    case user
    case admin
}
```

Keep the memberwise initializer internal so the signer is the only thing that can produce an
identity. Map every property to its registered claim name through `CodingKeys`. This is a wire
contract between the service that mints tokens and every service that verifies one: add claims
under new keys, and never repurpose an existing key.

Carry `sub` as `String` rather than `UUID` when service accounts need a subject that is not a user
identifier.

## What belongs in the token

A token answers *who is calling*. It never answers *what they may do*. Carry the authentication
claims and the identity attributes a service needs in order to decide for itself — the subject, the
issuer, the validity window, and a role. Keep permissions, scopes, entitlements and feature grants
out of it.

The distinction is not stylistic. A role is a fact about a user, owned by the service that stores
users. A permission is a policy, owned by whichever service enforces it. Putting
`scopes: ["users:list", "billing:refund"]` in the token moves every service's policy into the one
service that mints tokens: adding an RPC now means changing the issuer, and the set of things a
token authorises can only be discovered by reading every consumer. Putting `role: admin` there
instead lets each service answer "may an admin do this *here*?" in its own code, beside the handler
that does it.

Every claim is a copy of state that can go stale. It is fixed at signing and only changes when the
token is refreshed, so a demotion takes effect up to one access-token lifetime late. That is the
same bound that already applies to a revoked session, and it is why the access-token expiry is kept
short. It is also why the claim set should stay small: a claim nothing reads is a liability that
looks like a control, and a claim everything reads is a cache nothing invalidates.

## Signing

```swift
public struct IdentitySigner: Sendable {
    public struct Configuration: Sendable {
        public let issuer: String
        public let privateKey: EdDSA.PrivateKey
    }

    public func makeIdentity(subject: String, role: UserRole, expiration: TimeInterval) -> Identity
    public func signIdentity(_ identity: Identity) async throws -> String
}
```

The caller states who the token is for, what role they hold, and how long it lasts. `role` is the
caller's to state because only the caller has looked the subject up: the signer attests a claim, it
does not know what is true of a user. `iss` and `iat` belong to the signer: an issuer is a property
of the service doing the signing rather than of any one token, so stating it per call site is the
same value repeated everywhere and wrong wherever it drifts.

Return the identity rather than signing in one step. A caller that mints a token almost always has
to report when it expires, and deriving that expiry a second time leaves the token and what the
caller says about it as two readings of the clock that can disagree:

```swift
let identity = signer.makeIdentity(
    subject: user.id.uuidString,
    role: user.role,
    expiration: policy.accessTokenExpiration
)
let token = try await signer.signIdentity(identity)

return Session(accessToken: token, accessTokenExpirationDate: identity.expiration.value)
```

Keep `Configuration` holding typed key values, never PEM strings. The library should not be in the
business of sniffing key encodings, and a parsed key does not linger in memory as recoverable text.

## Identifying a caller versus requiring one

Two separate decisions. Identifying is infrastructure and belongs in the shared package; requiring
is policy and belongs to the service.

- The identifying interceptor binds the caller when a token is present and leaves the context empty
  when there is none. Apply it broadly, to every RPC.
- Requiring a caller is the handler's job: it reads the task local and refuses a `nil` one.

On HTTP, Hummingbird ships both halves already, so the split is `AuthenticatorMiddleware` and
`IsAuthenticatedMiddleware`. Do not assume gRPC wants the mirror image. A server interceptor is
dispatched on `MethodDescriptor` and can reach the request body only by consuming
`request.messages` and re-emitting it, so it can express "an admin may call this RPC" but not "your
own record, or any record if you are an admin". The second kind is most of real authorization and
has to live in the handler anyway; an enforcing interceptor added early leaves two mechanisms
maintained for one decision.

Write the check in the handler until three or more RPCs need the same purely static role rule. Only
then lift it into a `RoleServerInterceptor(allowedRoles:)` in the shared package's gRPC product,
applied per method in the composition root so the whole policy reads in one place. Have it read the
already-bound identity rather than verifying the token a second time, and have it refuse an
anonymous caller rather than passing one through.

## Authorization lives in the service

The token supplies the role. Each service decides what that role may do inside it, in its own code,
next to what it protects.

```swift
private func requireAdmin() throws {
    guard let identity = IdentityContext.current?.identity else {
        throw RPCError(code: .unauthenticated, message: "Authentication is required.")
    }
    guard identity.role == .admin else {
        throw RPCError(code: .permissionDenied, message: "Administrator access is required.")
    }
}
```

Distinguish the two failures. A missing caller is `.unauthenticated`, because presenting a token
could change the answer. A caller whose role is insufficient is `.permissionDenied`, because
presenting a different token could not. Collapsing them tells a client to go and refresh a token
that was never the problem, and it will keep refreshing.

Gate the RPCs that answer about someone other than the caller. Leave open the ones the
authenticating service reaches on behalf of a caller who cannot yet prove who they are: registering
creates a user and logging in looks one up, both before any token exists. An authorization rule
that breaks the login path is the one failure a client cannot recover from by logging in again.

Define the role enum once, in the shared identity package, because every verifying service reads
the claim and a copy per service is the same contract restated once per reader. A service that also
persists roles keeps its own domain enum and maps at the transport boundary, the way it maps every
other message. Refuse a role the wire does not name rather than reading it as the least privilege:

```swift
case .unspecified, .UNRECOGNIZED:
    throw UserServiceError.unknown
```

`unspecified` is an unset field and `UNRECOGNIZED` is a value added to the contract after this
service was built. Both mean the sender said something this service cannot interpret. Reading
either as `user` looks like a working login while silently stripping an administrator of their
role — far harder to notice than a login that stops.

Absent and invalid are not the same thing. A caller who presents nothing has claimed nothing; a
caller whose token does not verify has made a claim that failed. Pass the first through
anonymously and refuse the second with `unauthenticated`. Reading an invalid token as anonymous
turns an expired token into a silent loss of privileges on an unprotected RPC and hides a
misconfigured client whose credentials nothing ever looks at.

## Exclude the RPCs that issue a session

The RPCs that hand out a session — password login, token refresh — are reached precisely when the
caller's current session is no good. Excluding them from identification is required, not
optional:

```swift
GRPCServer(
    transport: …,
    services: [authenticationService],
    interceptorPipeline: [
        .apply(
            IdentityServerInterceptor(verifier: verifier),
            to: .allExcluding(services: [], methods: AuthenticationService.publicMethods)
        )
    ]
)
```

A client that attaches its access token to every outgoing call — which is what the client
interceptor below does — will send its expired token along with the refresh request. Refusing a
present-but-invalid token then refuses the one call that could replace it, and the client stays
locked out until something strips the header. Build the excluded set in the transport target from
the generated `Method.<Name>.descriptor` statics so it cannot drift from the proto, and expose it
as a named `Set<MethodDescriptor>` rather than spelling descriptors in the composition root.

## Propagating the caller

`ServerContext` carries the method descriptor and the peers, not the request metadata, so a handler
has no way to reach the token it arrived with. Bind both to a task local:

```swift
public struct IdentityContext: Sendable {
    public let identity: Identity
    public let token: String

    @TaskLocal public static var current: IdentityContext?

    public static func withIdentity<T>(
        _ identity: Identity,
        token: String,
        isolation: isolated (any Actor)? = #isolation,
        operation: () async throws -> T
    ) async rethrows -> T
}
```

A client interceptor reads it back and re-attaches the token to outgoing calls, so one token
identifies the caller at every service in the chain. Forward the token unchanged; never reissue it,
since only the signing service can produce another. Register the interceptor on the `GRPCClient`
rather than per call, so a service cannot forget it.

A call made outside a caller's request — startup work, a workflow Activity, a scheduled job — goes
out unauthenticated rather than failing. Whether that is acceptable belongs to the receiving
service, which says so by choosing which RPCs it protects. The corollary is real: an RPC the
authenticating service calls before a caller exists must be reachable anonymously.

## Reading the credential

Parse `authorization` yourself rather than trusting a framework helper:

- Match the scheme without regard to case, as RFC 7235 defines it. Requiring exactly `Bearer`
  reads a `bearer` header — which grpc-web clients and proxies send — as no credential at all, and
  an unauthenticated caller is far harder to notice than a rejected one.
- Take the first `authorization` entry, not the first that happens to parse. Taking the latter lets
  a caller hide a second credential behind one the service ignores.
- Use `replaceOrAddString` when attaching a token. A second entry leaves which one the receiver
  reads down to ordering.
- Use the same parser on every transport. Two transports disagreeing about what counts as a
  credential is a silent authorization difference.

## Key material in configuration

Keys reach a service base64 encoded, because a PEM's newlines do not survive an environment
variable. Decode in the composition root, and check for a PEM before attempting base64:

```swift
extension ConfigReader {
    func requiredPEM(forKey key: ConfigKey, isSecret: Bool = true) throws -> String {
        let configured = try requiredString(forKey: key, isSecret: isSecret)

        guard !configured.contains("-----BEGIN") else { return configured }
        guard let data = Data(base64Encoded: configured, options: .ignoreUnknownCharacters) else { return configured }

        return String(decoding: data, as: UTF8.self)
    }
}
```

The ordering is load-bearing. Base64 decoding tolerates every character a PEM is made of, so
decoding a raw PEM produces plausible-looking rubbish instead of failing, and whether it survives
depends on the document's length modulo four. Test the input, never the decoded output.

Give each configuration an `init(config:)` in the executable's `Configuration` folder, beside
`PostgresClient.Configuration+ConfigReader.swift`:

```swift
extension IdentitySigner.Configuration {
    init(config: ConfigReader) throws {
        try self.init(
            issuer: config.string(forKey: "issuer", default: "<project>-authentication"),
            privateKey: EdDSA.PrivateKey(pem: config.requiredPEM(forKey: "privateKey"))
        )
    }
}
```

Ship a `scripts/generate-keys.sh` with the issuing service that generates the pair into a temporary
directory, prints the two base64 lines, and fills `.env` on request. Generating into the working
tree invites committing a private key; generating into `mktemp -d` under a trap cannot.

## Deployment

Declare verification once and merge it into every service that verifies a token, including the
issuing service when it protects any RPC of its own:

```yaml
x-jwt-verification: &jwt-verification
  JWT_PUBLIC_KEY: ${JWT_PUBLIC_KEY:?Set JWT_PUBLIC_KEY in .env before starting the stack}

services:
  authentication:
    environment:
      <<: [*postgres-connection, *jwt-verification]
      JWT_PRIVATE_KEY: ${JWT_PRIVATE_KEY:?Set JWT_PRIVATE_KEY in .env before starting the stack}
```

Verify the anchor is actually merged. An unreferenced YAML anchor is silently ignored, so a
`${VAR:?message}` guard inside one never fires and the stack starts without a key the service
requires. `docker compose config <service>` shows what a service will really receive.

Rotating means generating a new pair and restarting every service. Access tokens signed by the old
key stop verifying and clients recover on their next refresh, provided refresh tokens are database
rows rather than signed tokens.

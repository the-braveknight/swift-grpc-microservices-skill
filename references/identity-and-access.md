# Identity and access

How a caller is identified across services, who may mint a token, and how the credential travels.

## The shared identity package

Put the claim payload, the signer, the verifier, and one adapter per transport in a shared
`<project>-identity` package beside `<project>-protos`, depended on by tagged URL. Products:

- `Identity` — the payload, the signer, the verifier, and the task local. Depends on JWTKit only.
- `IdentityGRPC` — the server and client interceptors. Depends on `Identity` and `GRPCCore`.
- `IdentityHTTP` — the authenticating middleware. Depends on `Identity` and Hummingbird.
- `ServiceIdentity` — a process's own identity: the session, its interceptor, and the adapter
  that speaks `IssueServiceToken`. Depends on `Identity`, `IdentityGRPC`, and the authentication
  contract from `<project>-protos` — the one product that does.

It is a separate package from `<project>-protos` because it is runtime behavior rather than a
contract, and separate products because a gRPC-only service must not link Hummingbird to verify a
token, and a service that only verifies must not link the authentication contract. Core may link
`Identity`; it is a focused shared package, not an infrastructure SDK.

Because it is consumed by tag, a source edit here is invisible to every service until it is tagged
and each consumer's `Package.resolved` is updated. Verify a cross-repo change before tagging by
pointing a consumer at the working copy:

```sh
swift package edit <project>-identity --path ../<project>-identity   # resolution must succeed first
swift build
swift package unedit <project>-identity
```

Edit mode needs a resolvable graph to enter and to leave. When the consumer's manifest already
names a tag that does not exist yet — or a package whose repository does not exist yet — resolution
fails and `edit` refuses. Temporarily rewrite the `.package(url:from:)` lines to `.package(path:)`,
build, and restore them; `swift package unedit` fails the same way, so revert the constraint, unedit,
`git checkout -- Package.resolved`, then re-apply the constraint.

Adding a case to a public enum here breaks every consumer that switches over it exhaustively. Plan
it as a breaking change across the services, even though a `from:` requirement will resolve the new
tag automatically and present it as a build failure rather than a resolution one. Adding a case to
`UserRole` breaks them differently and worse: a token carrying the new role fails to *decode* on a
verifier built against the old tag, so every RPC that token reaches is refused as `unauthenticated`.
Bump every verifying service to the new tag before the first token with the new role is minted.

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
    case service
}
```

Keep the memberwise initializer internal so the signer is the only thing that can produce an
identity. Map every property to its registered claim name through `CodingKeys`. This is a wire
contract between the service that mints tokens and every service that verifies one: add claims
under new keys, and never repurpose an existing key.

Carry `sub` as `String`, not `UUID`. A person's subject is their user id; a process's subject is
the name its credential was issued under, and both are one claim.

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

A call made outside a caller's request — startup work, a workflow Activity, a scheduled job — has
nothing to forward, and the forwarding interceptor sends it out unauthenticated rather than failing.
A process that should identify itself on such calls presents its own identity instead, through a
separate client — see *Processes* below. That includes the issuer: the authenticating service
reaches the users service before any caller exists — signing someone in, refusing a duplicate
registration — so it holds a credential of its own and exchanges it through its own
`IssueServiceToken`, over the network, exactly like every other process. It could sign for
itself instead, since it holds the key; do not. One mechanism for every process is worth one
credential and one round trip, and it keeps the issuer from being the one process that
authenticates differently from the rest.

## Processes: the service role

A worker, a service reacting to a provider's webhook, a reconciliation job — anything that calls
other services with no inbound request behind it — is a principal of a second kind. Give it its own
role, `service`, rather than lending it `admin`: an admin token works at the HTTP edge too, so a
leaked worker credential would be a full administrative login. With a role of its own, the gateway
refuses it on every route, and each service admits it per RPC exactly as it admits `admin`.

Its subject is the name its credential was issued under, not a user id. That is why a person-only
guard tests the **role**, never the shape of the subject:

```swift
private func userId(of identity: Identity) throws -> UUID {
    guard identity.role != .service, let userId = UUID(uuidString: identity.subject) else {
        throw RPCError(code: .permissionDenied, message: "This operation requires a user.")
    }
    return userId
}
```

A credential can be issued under any name, including one shaped like a UUID; a guard that only
parsed the subject would let such a process through as that user. Test the role first, then parse.
Do not instead forbid UUID-shaped credential ids at issue time as the sole defence — that protects
only the guards that exist today, and the next one written by copying the old pattern reopens it.

Which RPCs admit a process is the receiving service's decision, per RPC, like the admin check.
Granting and revoking an entitlement are what a payment does, so they take `admin` or `service`;
listing a user's own devices is what a person does, so `service` is refused:

```swift
private func requireAdminOrService() throws {
    switch try requireIdentity().role {
    case .admin, .service: return
    case .user: throw RPCError(code: .permissionDenied, message: "Administrator or service access is required.")
    }
}
```

The HTTP gateway is for people. Every route that acts on "the caller's own" account sends
`identity.subject` upstream as a user id, so give `IdentityRequestContext` a `requireUser()` that
refuses `role: service` with `403` before the subject leaves the process — otherwise a users service
reports the credential name as a malformed id, which is the wrong error for the right refusal.

## Service credentials and issuance

A process authenticates the way a person does with a password, and the issuer treats it the same
way. The authenticating service keeps a `ServiceCredential` — an id, the bcrypt digest of a
secret, a creation date — and exposes one RPC:

```proto
rpc IssueServiceToken(IssueServiceTokenRequest) returns (IssueServiceTokenResponse);

message IssueServiceTokenRequest  { string client_id = 1; string client_secret = 2; }
message IssueServiceTokenResponse { string access_token = 1; google.protobuf.Timestamp expiration_date = 2; }
```

Its own response type, not the session type: a process gets no refresh token — it holds its secret
and exchanges it again — and a session with an empty refresh field would make the contract lie.
It is a public method, excluded from identification like login, because it is how a process obtains
the token it would otherwise present. An unknown id and a wrong secret answer the same
`unauthenticated`, so nothing reaching the port can enumerate which service names exist.

Issue a credential with an operator command on the issuing service, never an RPC — issuing is what
lets a process obtain tokens, so it must not be reachable by anything holding one:

```sh
authentication service-credentials create --id payments-worker > secrets/payments-worker.secret
```

Mint the secret rather than accept one, store only its digest, print it once to standard output and
nothing else there, so a redirect captures exactly the secret. Validate the id as a name — non-empty,
no whitespace — and nothing more; the role guard above is what keeps a name out of person-only
operations. The credential lives in the issuer's database, so rotation is a row and a restart, not
a redeploy of the issuer, and it comes into being only after that database is migrated: the first
start of a stack is staged — infrastructure and the issuer, issue the credential, then everything.

Give the token its own lifetime, `JWT_SERVICE_TOKEN_EXPIRATION`, beside the session's. It is a
different trade-off: with no refresh token to delete, the token's expiry *is* how long revoking a
credential takes to bite. Keep it comfortably above the session's refresh window below — at or under
it, every call would refresh.

## Presenting a process's own identity

The consumer side lives in the identity package and knows nothing about the issuer's contract:

- `ServiceIdentityClient` — a protocol: credentials in, a token out, typed errors
  `invalidCredentials` / `unavailable` / `unknown`. The split is the point: refused credentials
  mean the process's standing is gone and no retry helps; an unreachable issuer says nothing about
  the credentials.
- `ServiceIdentitySession` — an actor holding the credentials and the token they last bought, with
  one `State`: `unauthenticated`, `authenticated(token)`, or `authenticating(task)`. `accessToken`
  returns the held token unless it is within five minutes of expiry, in which case it refreshes and
  the caller waits for the new one. `refresh()` joins an exchange already in flight rather than
  starting a second, which is what "one in flight" means; because several callers settle the same
  task in turn, only the exchange the session is still waiting on may record its outcome, or a late
  caller from a failed one would overwrite a newer exchange and the next call would start a third.
- `ServiceIdentityInterceptor` — a client interceptor that attaches the session's token to every
  call and, when the receiver refuses it as `unauthenticated` *at acceptance* (before any message
  crossed, so the request can be resent whole), refreshes once and resends. Once: a second refusal
  means the credential itself is no good. Its errors reach the caller through
  `RPCErrorConvertible`, which grpc-swift applies to whatever an interceptor throws — refused
  credentials as `unauthenticated`, an unreachable issuer as `unavailable`.

A client carrying this interceptor speaks as the process on every call, whatever identity is
bound at the time. A process that also forwards callers does so through a *separate* client
carrying the forwarding interceptor; registering both on one client would leave the header to
whichever ran last. One client per identity.

The conformance that speaks `IssueServiceToken` behind the protocol needs the authentication
contract. It lives in the `ServiceIdentity` product of the identity package, which is the one
product there that links `<project>-protos`: `Identity`, `IdentityGRPC` and `IdentityHTTP` stay
linkable without the contract, so a service that only verifies never pulls it in, and a process
that acts as itself links one product and gets the session, the interceptor and the adapter
together. Not the protos package, which is generated contract and nothing else; not the issuing
service's own package, which would drag its whole graph into every consumer; and not a fourth
package, which is one more tag to keep in step for one type.

In the composition root, build the session over the adapter, put the interceptor on the client that
speaks as the process, and exchange once at startup racing the service group — so a wrong secret or
an unreachable issuer fails the process there, naming the problem, rather than failing the first
activity minutes after the deploy looked fine. Configure the credential as `SERVICE_CREDENTIAL_ID`
and a `SERVICE_CREDENTIAL_SECRET_PATH` to the mounted file, trimming the trailing newline the
issuing command wrote.

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

Configuration carries the path to a key, not the key itself. The composition root opens the file:

```swift
extension IdentitySigner.Configuration {
    init(config: ConfigReader) throws {
        let privateKeyPath = try config.requiredString(forKey: "privateKeyPath")
        let privateKey = try String(contentsOfFile: privateKeyPath, encoding: .utf8)

        try self.init(
            issuer: config.string(forKey: "issuer", default: "<project>-authentication"),
            privateKey: EdDSA.PrivateKey(pem: privateKey)
        )
    }
}
```

Give each configuration an `init(config:)` in the executable's `Configuration` folder, beside
`PostgresClient.Configuration+ConfigReader.swift`.

A path is what the surrounding libraries already take. `TLSConfig.CertificateSource.file(path:format:)`
and `PrivateKeySource.file(path:format:)` in grpc-swift, and `NIOSSLCertificate.fromPEMFile` in
NIOSSL, are handed a path and open it themselves, so the mTLS material a service needs next is
configured exactly like the signing key it needs now. Nothing about a path resists an environment
variable, which is the whole reason the document used to be encoded.

It matters most for a private key. An environment variable is readable from `/proc/<pid>/environ`,
reported by the container runtime's inspect command, and inherited by every child process; a
mounted file is none of those. It also keeps the material from becoming a configuration value at
all, which is what an access reporter would otherwise be free to log.

Fail loudly when the file is unreadable, naming both the key and the path. An absent mount, a
wrong path, and the wrong file mode are different deployment mistakes with different fixes, and a
failure that names neither leaves the operator to guess.

If you inherit a system that carries the encoded document in the variable instead, the decoding
order is load-bearing: test the input for a `-----BEGIN` header *before* attempting base64, never
the decoded output. Base64 decoding tolerates every character a PEM is made of, so decoding a raw
PEM produces plausible-looking rubbish rather than failing, and whether it survives depends on the
document's length modulo four. Prefer migrating it to a path.

Ship a `scripts/generate-keys.sh` with the issuing service that writes the pair as PEM files and
refuses to overwrite an existing pair without `--force`, since rotating invalidates every access
token in flight. The private key now has to land on disk for a container to mount it, so protect
it there: create the directory `0700`, set `umask 077` *before* `openssl` writes so the key is
never briefly world-readable between creation and `chmod`, and add the directory to
`.gitignore`.

## Deployment

Mount each key as a deployment secret and give every service the path, not the document. Declare
verification once and merge it into every service that verifies a token, including the issuing
service when it protects any RPC of its own:

```yaml
x-jwt-verification: &jwt-verification
  JWT_PUBLIC_KEY_PATH: /run/secrets/jwt-public

secrets:
  jwt-public:
    file: ./secrets/jwt-public.pem
  jwt-private:
    file: ./secrets/jwt-private.pem

services:
  authentication:
    environment:
      <<: [*postgres-connection, *jwt-verification]
      JWT_PRIVATE_KEY_PATH: /run/secrets/jwt-private
    secrets:
      - jwt-public
      - jwt-private
```

Compose refuses to start when a secret's source file is missing, so a stack without keys fails at
`up` rather than at the first request — the same guarantee a `${VAR:?message}` guard gives a
variable.

Verify the anchor is actually merged. An unreferenced YAML anchor is silently ignored, so a
`${VAR:?message}` guard inside one never fires and the stack starts without a value the service
requires. `docker compose config <service>` shows what a service will really receive.

On a platform without compose secrets, the equivalent is a file mount plus the path variable. Note
that most will deploy a container whose mount is missing rather than refusing, so the failure
appears in the logs as an unreadable key instead of a failed deploy; check them after the first
rollout rather than reading a green deploy as proof the mount landed.

Rotating means generating a new pair and restarting every service. Access tokens signed by the old
key stop verifying and clients recover on their next refresh, provided refresh tokens are database
rows rather than signed tokens.

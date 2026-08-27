# Postgres persistence

Keep every Postgres type in `<Service>Postgres`, except executable configuration and client construction.

## Contents

- Database and context
- Repositories and statements
- Idempotent writes
- Database ownership and migrations
- The service role
- Row-level security
- Transaction policy

## Database and context

Define the context factory contract:

```swift
package protocol PostgresContext: Sendable {
    init(connection: PostgresConnection, logger: Logger)
}
```

Implement both database operations and unwrap transaction failures:

```swift
package struct PostgresDatabase<Context: PostgresContext>: Database<Context> {
    private let client: PostgresClient
    private let logger: Logger

    package init(client: PostgresClient, logger: Logger) {
        self.client = client
        self.logger = logger
    }

    package func withConnection<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T {
        try await client.withConnection { connection in
            try await operation(Context(connection: connection, logger: logger))
        }
    }

    package func withTransaction<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T {
        do {
            return try await client.withTransaction(logger: logger) { connection in
                try await operation(Context(connection: connection, logger: logger))
            }
        } catch let error as PostgresTransactionError {
            throw error.beginError
                ?? error.closureError
                ?? error.commitError
                ?? error.rollbackError
                ?? error
        }
    }
}
```

The service context conforms to `PostgresContext` and the exact use-case context protocols needed by its repositories:

```swift
package struct PostgresCatalogContext: PostgresContext, CreateItemUseCaseContext, ListItemsUseCaseContext {
    package let itemRepository: any ItemRepository

    package init(connection: PostgresConnection, logger: Logger) {
        self.itemRepository = PostgresItemRepository(
            connection: connection,
            logger: logger
        )
    }
}
```

Add repositories to this context as features are introduced. Do not expose the raw connection to Core.

## Repositories and statements

Use `PostgresPreparedStatement` values in `Statements/<Entity>`. Keep SQL and row decoding there. Repositories execute statements and translate database failures into `XRepositoryError`.

For duplicate detection, inspect the PostgreSQL server error code and the exact unique constraint. Map SQLSTATE `23505` for the known constraint to `.duplicateIdentifier`; map other failures to the appropriate repository failure. Do not let `PSQLError` escape into Core.

Use database-owned UUIDv7 identifiers and timestamps for Postgres records:

```sql
CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    name TEXT NOT NULL,
    creation_date TIMESTAMPTZ NOT NULL DEFAULT NOW()
)
```

The create command omits `id`. The create statement inserts caller-owned fields and returns the complete row, including the database-generated identifier:

```sql
INSERT INTO items (name)
VALUES ($1)
RETURNING id, name, creation_date
```

Decode identifiers as `UUID`, dates as `Date`, and return a Core entity from the repository. Do not put `id` in `CreateItemCommand`; do not put a timestamp there unless time is a true caller-supplied business value. Do not add an identifier-generator protocol or generate entity IDs in a use case, repository, workflow, RPC client, or another service.

Distinguish entity identifiers from non-identifier application values such as email-verification tokens and refresh tokens. Model those as one Core value type that owns both minting and digesting, rather than a generator protocol and a separate hashing helper:

```swift
package struct SecretToken: Equatable, Sendable {
    package let value: String
    package let digest: String

    package init(byteCount: Int = 32)          // mint from CSPRNG bytes, base64url
    package init?(presented value: String)     // wrap what a caller sent back
    private init(digesting value: String)      // both funnel here: value + SHA-256 hex
}
```

`digest` is then never something a call site must remember to derive. Do not add a generator type beside it; a concrete struct with no protocol cannot be substituted, so injecting it buys nothing, and its statics drag verify-only callers into depending on the minting type.

Persist only `digest`, hand `value` to its bearer once, and declare the column `TEXT UNIQUE NOT NULL` with no generation default. Use SHA-256 rather than a password hash: the input is already high entropy, and a deterministic digest is what makes lookup by index possible. Reserve bcrypt for user-chosen passwords. Delete the row when its process reaches a terminal state so no secret outlives the flow that issued it.

## Idempotent writes

Enforce retry safety where the side effect is owned. A caller or workflow supplies one stable, namespaced `idempotencyKey` for a logical mutation; the receiving use case validates it, the command and statement carry it, and the owning database enforces it. Do not rely on an in-memory check, a client-side retry flag, or Temporal history to prevent duplicate writes.

For an idempotent create, put a single-column `UNIQUE` constraint on the column declaration and make conflict handling atomic. PostgreSQL gives the inline constraint the conventional `<table>_<column>_key` name, which may be referenced by `ON CONFLICT`:

```sql
CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    idempotency_key TEXT UNIQUE,
    name TEXT NOT NULL,
    creation_date TIMESTAMPTZ NOT NULL DEFAULT NOW()
)
```

Keep the column nullable only when explicitly trusted admin, import, or migration paths may create records without retry identity. PostgreSQL's ordinary unique constraint permits multiple `NULL` values. Require a non-empty, length-bounded key at every RPC or workflow-backed write boundary that promises idempotency; a `NULL` write has no replay guarantee.

```sql
INSERT INTO items (idempotency_key, name)
VALUES ($1, $2)
ON CONFLICT ON CONSTRAINT items_idempotency_key_key DO UPDATE
SET idempotency_key = EXCLUDED.idempotency_key
WHERE items.name = EXCLUDED.name
RETURNING id, name, creation_date
```

Compare every canonical field that defines the operation, after the service's normal normalization. The same key and input returns the existing database-generated result. The same key with different input returns no row and maps through a repository `.idempotencyConflict` to a use-case conflict. A different key that violates a business constraint remains a normal duplicate error. Never overwrite stored business data with a mismatched retry.

Use compare-and-swap updates for retryable state transitions and stable provider idempotency keys for external side effects such as email or payments. If deleting the result would allow a key to be reused but the product requires longer retention, store operation outcomes in a dedicated idempotency table instead of tying key lifetime to the entity row.

Guard a state transition in the `WHERE` clause and let the absent row report the loss:

```sql
UPDATE registrations
SET state = $2, update_date = NOW()
WHERE id = $1 AND state = $3
RETURNING id, state, creation_date, expiration_date
```

Returning no row is how a caller learns it lost the race, atomically and without a separate read.

Carry only the destination state in the command. When each transition has one legal predecessor, derive the guard from the destination on the state enum instead of passing both: a second field carries no information the destination does not already imply, only the chance to pass an inconsistent pair. Give the initial state no predecessor so it binds as SQL `NULL`, never matches, and fails closed.

For a business value that must be unique only while a process is live — such as one in-flight registration per email — enforce it with a partial unique index over the active states, named `<table>_active_<column>_key`:

```sql
CREATE UNIQUE INDEX registrations_active_email_key
ON registrations (email)
WHERE state IN ('pending', 'verified')
```

Rows in terminal states fall out of the index, so the value frees automatically when the process completes or expires. Scope the predicate to states this service owns the truth about; after the process hands ownership to another service, that service's own constraint is the authority. Map SQLSTATE `23505` on the named index to the matching repository error. Never replace this constraint with a check-then-insert in application code, and never convert the conflict into success by looking up the business value; distinct principals can submit identical business data.

## Database ownership and migrations

An extracted microservice owns the whole database. Use unqualified names such as `items`, not `<service>.items`, and do not create a service-named schema.

Name migrations for their result, such as `CreateItemsTable`. Do not prefix a migration with the service name because the migration already lives inside the service-owned Postgres module. Give each table its own create migration; keep that table's indexes and constraints with it rather than combining several tables into one create migration. Keep migrations under `Migrations/<Entity>` and register them explicitly in dependency order in the executable `database migrate` command, with referenced parent tables before child tables.

Write single-column uniqueness inline, such as `email TEXT NOT NULL UNIQUE`; use table-level `UNIQUE (...)` only for multi-column uniqueness. Do not add `CHECK (... IN (...))` constraints unless the user explicitly requests them.

Name stored dates `creation_date`, `update_date`, `expiration_date`, `consumption_date`, and equivalent noun-based names. Use the matching Swift camel-case forms `creationDate`, `updateDate`, `expirationDate`, and `consumptionDate`. Do not use `created_at`, `expires_at`, `expiry_date`, or Swift `somethingAt` names. Noun-based names describe the stored value rather than the event, and they convert mechanically between SQL snake case and Swift camel case with no special cases.

Do not delete the monolith's table or migration during the copy phase. Complete producer and consumer verification, decide how existing data moves, execute the cutover, and only then remove old persistence. Schema/table deletion and data migration are destructive product decisions; ask when the desired strategy is not stated.

## The service role

Every service has two Postgres roles, and its **first migration** creates the second one:

- the owner — `POSTGRES_USER` — runs `database migrate` and owns every table;
- the service role — `<service>_service`, from `POSTGRES_SERVICE_ROLE` / `POSTGRES_SERVICE_PASSWORD`
  — is what `serve`, `worker`, and every other command that reads or writes data connect as. It
  may read and write the service's tables and nothing more.

```swift
let migrations = DatabaseMigrations()
await migrations.add(
    CreateServiceRole(role: serviceRole, password: servicePassword, database: database)
)
await migrations.add(CreateUsersTable())
```

This is the standard whether or not the service confines rows with policies. Postgres applies no
policy to a table's owner, so a service that runs as the owner cannot add row-level security later
without also changing what it connects as; a service that has run as its service role from the
first migration adds a policy migration and nothing else.

The migration is plain: `CREATE ROLE "<role>" LOGIN PASSWORD '…'`, `GRANT CONNECT` on the
database, `GRANT USAGE` on `public`, DML on all tables and usage on all sequences, and the same two
as `ALTER DEFAULT PRIVILEGES` so every table a later migration creates is the role's from the moment
it exists. `revert` is `DROP ROLE IF EXISTS`. Keep it that simple — no existence checks, no
quoting helpers; the values are the deployment's own configuration. Roles are cluster-wide, so a
database dropped and re-migrated in a cluster that still has the role fails on `CREATE ROLE`;
drop the role with the database.

The migration library refuses a list whose order differs from what is applied — it throws, it does
not revert — so a service that adopts this after its tables are applied cannot slide the role in
first without re-migrating from scratch. Adopt it at the first migration.

## Row-level security

When a service's rows belong to callers — a user's entitlements, a user's devices — confine callers
in Postgres, not in the statements. Restating the rule as a scope bound into every query is the
same predicate maintained twice, and the copy in the statements is the one that drifts. The rule
exists once, as policies; the service tells the database who is calling. The policies are enforced
against the service role above, which is why it exists before any of them.

**Stamp the transaction, not the connection.** The database learns who is calling from two
transaction-local settings the policies read:

```swift
if let identity = IdentityContext.current?.identity {
    try await connection.query(
        "SELECT set_config('app.caller_role', \(identity.role.rawValue), true)", logger: logger
    )
    if identity.role != .service, let userId = UUID(uuidString: identity.subject) {
        try await connection.query(
            "SELECT set_config('app.caller_user_id', \(userId.uuidString.lowercased()), true)", logger: logger
        )
    }
}
```

`set_config(…, true)` takes bound parameters, unlike `SET LOCAL`, so nothing is spliced into SQL,
and it scopes the value to the transaction, so a pooled connection carries nothing to its next user.
Name the settings for the *caller*, not a user: a caller need not be one. Set the role always and
the user id only for a person — a process's subject is a credential name, and the policy casts the
setting to `uuid`, so setting it to `payments-worker` fails the worker's every query. With no
identity bound, nothing is set and the policies admit no rows.

**The policies** admit a process beside an administrator and keep `NULLIF`:

```sql
CREATE POLICY user_isolation ON devices
    USING (
        current_setting('app.caller_role', true) IN ('admin', 'service')
        OR user_id = NULLIF(current_setting('app.caller_user_id', true), '')::uuid
    )
    WITH CHECK (
        current_setting('app.caller_role', true) IN ('admin', 'service')
        OR user_id = NULLIF(current_setting('app.caller_user_id', true), '')::uuid
    )
```

`NULLIF` is load-bearing: once a custom setting has been set on a connection, reading it after that
transaction yields `''` rather than `NULL`, and `''::uuid` is an error rather than a non-match. A
table nobody writes as a user — an entitlement is granted by a payment or an administrator — has a
`WITH CHECK` on the role alone. A join table is reachable through its parent: an enrollment's policy
is `EXISTS (SELECT 1 FROM entitlements e WHERE e.id = entitlement_id AND e.user_id = …)`, and the
subquery runs under the parent's own policy.

**Every operation is a transaction.** The stamp lives on the transaction, so a read outside one sees
no rows. Drop `withConnection` from the service's `Database` protocol rather than leave a method that
looks cheaper and silently opens a transaction; the single-operation rule in *Transaction policy*
does not apply to a service that confines with policies.

**The service still decides what RLS cannot say.** A policy filters; it cannot answer
`permissionDenied`. An RPC that names a user in the request — claim for this user, list this user's
entitlements — still checks in the handler that the named user is the caller or the caller is
privileged, and the person-only RPCs still refuse a process by role. What the policies replace is the
*confinement* of id-only RPCs: an entitlement or device belonging to somebody else lists as empty and
deletes as nothing, exactly as if it did not exist, so an id cannot be probed.

**Which services.** A service whose rows belong to callers — entitlements and devices, users,
customers and subscriptions in a payments service. A table with no owner can still take a
*read* policy: newsletter subscribers are anyone's to add and an administrator's or a process's
to list, so `FOR INSERT WITH CHECK (true)` beside `FOR SELECT USING (role IN ('admin','service'))`.
Not the authenticating service, whose rows are credential material looked up *by secret* on
anonymous paths (a refresh token by digest, a registration by email): a `user_id` policy there
breaks refresh for everyone, and the process is the trusted issuer. Not a public catalogue.

**`RETURNING` is a read.** Postgres applies the `SELECT` policy to the row an `INSERT … RETURNING`
hands back, and a caller the policy excludes gets `new row violates row-level security policy`,
not the row. An insert made by a caller who may not read — the anonymous subscribe — must not
`RETURNING`, and the RPC then answers with acceptance rather than the record. Change the contract
to say so rather than stamping the record's dates in the service.

**Verify as the service role.** Run the service against the role the policies apply to and
probe with three tokens — a user, another user, an administrator — plus a process. As the owner,
every case passes with the policies off, which proves nothing. Check the policy text itself too
(`pg_policies`), since a renamed setting in code and an unrenamed one in a migration already applied
fails only at query time.

## Transaction policy

- Use `withConnection` for one create, update, delete, get, or list repository operation, unless
  the service confines rows with row-level security, in which case every operation is a stamped
  transaction and `withConnection` does not exist.
- Use `withTransaction` when a use case must make multiple local repository mutations atomically.
- Build every repository in a transaction context from the same supplied connection.
- Never call another microservice from inside `withTransaction`.
- Never make two service databases participate in one transaction.
- For a multi-service workflow, use orchestration and explicit failure/retry semantics. Add an outbox only when asynchronous delivery requirements call for it.

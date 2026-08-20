# Postgres persistence

Keep every Postgres type in `<Service>Postgres`, except executable configuration and client construction.

## Contents

- Database and context
- Repositories and statements
- Idempotent writes
- Database ownership and migrations
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

## Transaction policy

- Use `withConnection` for one create, update, delete, get, or list repository operation.
- Use `withTransaction` when a use case must make multiple local repository mutations atomically.
- Build every repository in a transaction context from the same supplied connection.
- Never call another microservice from inside `withTransaction`.
- Never make two service databases participate in one transaction.
- For a multi-service workflow, use orchestration and explicit failure/retry semantics. Add an outbox only when asynchronous delivery requirements call for it.

# Postgres persistence

Keep every Postgres type in `<Service>Postgres`, except executable configuration and client construction.

## Contents

- Database and context
- Repositories and statements
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

Prefer database-owned timestamps for Postgres records:

```sql
CREATE TABLE items (
    identifier TEXT PRIMARY KEY,
    creation_date TIMESTAMPTZ NOT NULL DEFAULT NOW()
)
```

The create statement inserts the command fields and returns the complete row:

```sql
INSERT INTO items (identifier)
VALUES ($1)
RETURNING identifier, creation_date
```

Decode dates as `Date` and return a Core entity from the repository. Do not put a timestamp in `CreateItemCommand` unless time is a true caller-supplied business value.

## Database ownership and migrations

An extracted microservice owns the whole database. Use unqualified names such as `items`, not `<service>.items`, and do not create a service-named schema.

Name migrations for their result, such as `CreateItemsTable`. Do not prefix a migration with the service name because the migration already lives inside the service-owned Postgres module. Keep migrations under `Migrations/<Entity>` and register them explicitly and in order in the executable `database migrate` command.

Do not delete the monolith's table or migration during the copy phase. Complete producer and consumer verification, decide how existing data moves, execute the cutover, and only then remove old persistence. Schema/table deletion and data migration are destructive product decisions; ask when the desired strategy is not stated.

## Transaction policy

- Use `withConnection` for one create, update, delete, get, or list repository operation.
- Use `withTransaction` when a use case must make multiple local repository mutations atomically.
- Build every repository in a transaction context from the same supplied connection.
- Never call another microservice from inside `withTransaction`.
- Never make two service databases participate in one transaction.
- For a multi-service workflow, use orchestration and explicit failure/retry semantics. Add an outbox only when asynchronous delivery requirements call for it.

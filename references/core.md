# Core

## Contents

- Database boundary
- Feature structure

## Database boundary

Keep this protocol in `<Service>Core/Database/Database.swift`:

```swift
package protocol Database<Context>: Sendable {
    associatedtype Context: Sendable

    func withConnection<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T

    func withTransaction<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T
}
```

Use `withConnection` for one repository call. Use `withTransaction` only when two or more persistence operations form one atomic business action. Do not default reads or single writes to transactions.

The exception is a service that confines callers with row-level security. There the caller's identity is stamped on the transaction and the policies read it from there, so a read outside a transaction sees no rows; that service's protocol declares `withTransaction` alone, with a comment saying why there is deliberately no cheaper entry point. See *Row-level security* in persistence.md.

Do not hold a transaction across a remote call. The connection is pooled, so a slow dependency becomes pool exhaustion and one service's latency spike takes this service down with it.

The narrow exception is rotating a single-use secret: consume the old row, call the dependency, insert the replacement. If the call instead runs after the commit, a dependency outage destroys a credential whose validity that dependency has nothing to do with — a brief blip logs out every caller that happens to rotate during it. Take the exception only when the remote call is one fast read, the transaction is short, the alternative is destroying a caller's credential, and the call carries a deadline well under the pool's wait time. Set that deadline explicitly; without one the pool is bounded by the dependency's worst case.

## Feature structure

Model an entity as an immutable `Equatable, Sendable` struct when domain equality is useful. Keep simple values simple:

```swift
package struct Item: Equatable, Sendable {
    package let identifier: String
    package let creationDate: Date

    package init(identifier: String, creationDate: Date) {
        self.identifier = identifier
        self.creationDate = creationDate
    }
}
```

Keep write inputs to repositories as commands:

```swift
package struct CreateItemCommand: Sendable {
    package let identifier: String

    package init(identifier: String) {
        self.identifier = identifier
    }
}
```

The repository protocol expresses persistence operations and throws repository errors:

```swift
package protocol ItemRepository: Sendable {
    func create(_ command: CreateItemCommand) async throws -> Item
    func list() async throws -> [Item]
}
```

Create one narrow context protocol per use case:

```swift
package protocol CreateItemUseCaseContext: Sendable {
    var itemRepository: any ItemRepository { get }
}
```

Keep use-case construction generic over `DatabaseType`, expose it through `XUseCaseProtocol`, and use typed throws:

```swift
package struct CreateItemUseCase<DatabaseType>: CreateItemUseCaseProtocol where DatabaseType: Database, DatabaseType.Context: CreateItemUseCaseContext {
    private let database: DatabaseType
    private let logger: Logger

    package init(database: DatabaseType, logger: Logger) {
        self.database = database
        self.logger = logger
    }

    package func callAsFunction(
        input: CreateItemUseCaseInput
    ) async throws(CreateItemUseCaseError) -> Item {
        guard !input.identifier.isEmpty else {
            throw .invalidIdentifier
        }

        do {
            let item = try await database.withConnection { context in
                let command = CreateItemCommand(identifier: input.identifier)
                return try await context.itemRepository.create(command)
            }
            logger.info("Item created", metadata: ["itemId": "\(item.id)"])
            return item
        } catch ItemRepositoryError.duplicateIdentifier {
            logger.warning("Item create rejected: duplicate identifier", metadata: ["identifier": "\(input.identifier)"])
            throw .duplicateIdentifier
        } catch {
            throw .unknown
        }
    }
}
```

Inject a `Logger` into every use case that performs a meaningful mutation or a notable failure, and log the domain event at the boundary: `info` on the success path (after the write commits, before returning), `warning` on a known/recoverable failure branch (not-found, duplicate, rejected), `error` only for a genuine failure. Attach the identifiers that make a line searchable as metadata (`itemId`, `userId`, a correlation id) — never secrets, tokens, or full payloads. A pure list/get use case takes no logger unless it warns on not-found. This is the swift-log facade — permitted in Core by governing rule 3 — while the concrete Loki/stdout handlers are wired only in the composition root ([composition.md](composition.md)); the composition root passes its `logger` into each use case's initializer.

Do not pass `Date`, a clock, or a `now` closure into this use case merely to stamp a record. The repository owns that persistence concern.

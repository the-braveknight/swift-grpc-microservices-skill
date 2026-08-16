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

    package init(database: DatabaseType) {
        self.database = database
    }

    package func callAsFunction(
        input: CreateItemUseCaseInput
    ) async throws(CreateItemUseCaseError) -> Item {
        guard !input.identifier.isEmpty else {
            throw .invalidIdentifier
        }

        do {
            return try await database.withConnection { context in
                let command = CreateItemCommand(identifier: input.identifier)
                return try await context.itemRepository.create(command)
            }
        } catch ItemRepositoryError.duplicateIdentifier {
            throw .duplicateIdentifier
        } catch {
            throw .unknown
        }
    }
}
```

Do not pass `Date`, a clock, or a `now` closure into this use case merely to stamp a record. The repository owns that persistence concern.

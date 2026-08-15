# Core and tests

## Contents

- Database boundary
- Feature structure
- Test doubles
- Swift Testing conventions

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

Model an entity as an immutable `Equatable, Sendable` struct when equality is useful in tests. Keep simple values simple:

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

## Test doubles

Do not create a production `<Service>MemoryStore` or `<Service>InMemory` target solely to make Core testable. Implement a lightweight `Database` test double inside `<Service>CoreTests`:

```swift
struct MockDatabase: Database<MockCatalogContext> {
    let context: MockCatalogContext

    init(itemRepository: any ItemRepository) {
        self.context = MockCatalogContext(itemRepository: itemRepository)
    }

    func withConnection<T: Sendable>(
        _ operation: @Sendable (MockCatalogContext) async throws -> T
    ) async throws -> T {
        try await operation(context)
    }

    func withTransaction<T: Sendable>(
        _ operation: @Sendable (MockCatalogContext) async throws -> T
    ) async throws -> T {
        try await operation(context)
    }
}

struct MockCatalogContext: CreateItemUseCaseContext, ListItemsUseCaseContext {
    let itemRepository: any ItemRepository
}
```

This mock intentionally makes both database methods execute immediately against the supplied context; it is a fast use-case seam, not a transaction-behavior simulator. Put repository actors with deterministic return values or injected errors in the relevant test file. Add a production in-memory adapter only when the shipped application genuinely needs in-memory persistence, never as a default architecture target.

## Swift Testing conventions

Core test files import only Foundation when needed, `<Service>Core`, and `Testing`. Do not use `@testable import`; `package` access is available to package tests.

Use this shape:

```swift
import Foundation
import CatalogCore
import Testing

@Suite
struct CreateItemUseCaseTests {
    @Test
    func createsItem() async throws {
        let repository = MockItemRepository()
        let useCase = CreateItemUseCase(
            database: MockDatabase(itemRepository: repository)
        )
        let input = CreateItemUseCaseInput(identifier: "item-1")

        let item = try await useCase(input: input)

        #expect(item.identifier == input.identifier)
    }

    @Test
    func mapsDuplicateIdentifierRepositoryError() async throws {
        let repository = MockItemRepository(
            error: ItemRepositoryError.duplicateIdentifier
        )
        let useCase = CreateItemUseCase(
            database: MockDatabase(itemRepository: repository)
        )

        try await #require(throws: CreateItemUseCaseError.duplicateIdentifier) {
            try await useCase(input: CreateItemUseCaseInput(identifier: "item-1"))
        }
    }
}
```

Prefer exact typed error values with `#require(throws:)`. Do not bind `let error = ...` merely to compare it afterward. For a direct returned collection, prefer `#expect(try await useCase() == expected)`.

Keep the mock repository private at the bottom of the use-case test file when it is specific to that suite. Share only `MockDatabase` when multiple suites use the same database wrapper. Arrange values, insert one blank line, perform the action, insert one blank line, assert.

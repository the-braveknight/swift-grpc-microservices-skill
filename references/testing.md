# Testing

How a service's behavior is verified: the test target, the mocks, and the line between what a use-case test covers and what the edge and the database already enforce. Tests here run with no infrastructure — no Postgres, no network, no identity scaffolding — because the architecture puts those concerns where a unit test does not have to simulate them.

## Contents

- The test target
- Mocks
- The scoped database helper
- What a use-case test covers
- What a use-case test does not cover
- Porting tests from a monolith

## The test target

Every service package carries one test target, `<Service>CoreTests`, registered in the manifest with the Core target and the `swift-log` facade as its only dependencies:

```swift
.testTarget(
    name: "CatalogCoreTests",
    dependencies: [
        "CatalogCore",
        .product(name: "Logging", package: "swift-log"),
    ]
),
```

Tests are swift-testing — `@Suite`, `@Test`, `#expect`, `#require` — never XCTest. Because targets of one package use `package` access (see *Package, targets, and naming* in SKILL.md), the test target imports Core plainly; `@testable` is never needed, and needing it is a sign a symbol has the wrong access level.

Suites are named for the feature (`@Suite("Subscriber use cases")`), test functions for the behavior, and the display string states the expectation as a sentence: `@Test("duplicate repository errors become create use-case errors")`.

## Mocks

Mocks live in a `Mocks/` directory inside the test target, one file per concern, and mirror the shapes Core declares:

- **A mock repository is an actor** holding an array of entities and an optional injected failure, conforming to the Core repository protocol. Reads return the array; writes append a deterministic entity or throw the injected repository error. State lives behind the actor so concurrent tests cannot race it.
- **`MockDatabase` conforms to the Core `Database` protocol** and runs the operation against a held context with no transaction semantics:

```swift
struct MockDatabase<Context: Sendable>: Database {
    let context: Context

    func withTransaction<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T {
        try await operation(context)
    }
}
```

- **A mock context** is a struct conforming to every per-use-case context protocol the suite exercises, holding the mock repositories.

Dates in mocks are fixed (`Date(timeIntervalSince1970:)`), never `Date()`: an assertion against a mock's output must be reproducible.

Like the `Database` wrapper itself, these mocks are duplicated per service, never extracted into a shared testing package — the same *Duplicate the small shapes* rule that governs the production code (see architecture.md).

## The scoped database helper

Construct the mock database through one scoped helper beside the mocks, and write test bodies inside it:

```swift
func withDatabase<T>(
    repository: any SubscriberRepository,
    operation: (MockDatabase<MockContext>) async throws -> T
) async rethrows -> T {
    let database = MockDatabase(context: MockContext(subscriberRepository: repository))
    return try await operation(database)
}
```

A returning factory would be equivalent today; the scoped shape is chosen for where it goes: a later integration variant that provisions a real ephemeral database has the same signature, and no call site changes. `rethrows` keeps the call honest — a test whose closure cannot throw calls it without `try`.

## What a use-case test covers

Test the decisions the use case owns, through its public surface, against mocks alone:

1. **The success path**: calling the use case mutates or reads the repository as specified, and the returned shape carries the expected values.
2. **Every guard**: an input the use case refuses before I/O throws the use case's own typed error, and the repository was never reached (`#expect(subscribers.isEmpty)` after the refusal).
3. **Every error translation**: each named repository error the use case maps becomes the documented use-case error — inject the failure into the mock, capture with `#expect(throws:)`, compare the typed value.

That enumeration is the whole matrix: one test per case of the use case's error enum, plus the success path. A use case with typed throws makes the matrix checkable against the enum's cases.

## What a use-case test does not cover

- **Identity.** Requiring a caller is the handler's per-RPC decision and never lives in a use case (see *Authorization lives in the service* in identity-and-access.md) — so use-case tests need no token, no task-local, no interceptor scaffolding. If a use-case test wants an identity fixture, the check is in the wrong layer.
- **Row confinement.** Row-level security is enforced by the database from the stamped transaction (persistence.md); a mock database cannot exercise it and must not pretend to. Policies are verified against a real database as part of the persistence completion gates, not in unit tests.
- **Transport.** Status-code mapping belongs to the producer adapter and is exercised at the contract boundary, not by driving a gRPC server in a unit test.

The result is that `swift test` runs the whole target in milliseconds with no containers, which is what allows CI to run it on every pull request (delivery.md).

## Porting tests from a monolith

When a service is extracted from a modular monolith (extraction-runbook.md), its module tests port almost verbatim: the mocks and suites copy over, and only the seams that moved change. Domain-logic tests keep their assertions; tests that exercised in-process module calls across what is now a network boundary re-aim their mocks at the consumer's use-case protocol instead. Expect small renames — the extracted service's use cases follow the standard naming grammar — and expect the ported suite to gain tests for guards the extraction introduced.

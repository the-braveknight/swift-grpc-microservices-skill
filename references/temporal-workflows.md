# Temporal Workflows

Use this reference whenever a service adds or changes Temporal Workflows, Activities, clients, signals, queries, timers, or workers.

## Contents

- Module boundaries
- Workflow design
- Activity design and retries
- Workflow clients
- Transactions and durability
- Signals, queries, and state
- Timers and cancellation
- Worker composition
- Naming and file style

## Module boundaries

Add `<Service>Workflows` only when a capability requires durable waiting, retries, timers, or multi-step orchestration. Use this dependency direction:

```text
<Service>Core ← <Service>Workflows ← <Service>
```

Keep these types in Core:

- workflow-client protocols used by use cases;
- workflow state and result values;
- Activity service protocols and their typed errors;
- domain inputs and outputs used by Activities.

Keep these types in `<Service>Workflows`:

- `@Workflow` definitions;
- `@ActivityContainer` definitions;
- `TemporalXWorkflowClient` adapters;
- Temporal-specific options and error translation.

Core must not import Temporal. The Workflows target must not implement SQL, construct provider clients, read configuration, or own executable lifecycle.

## Workflow design

Use Workflow code only for deterministic orchestration. Call `WorkflowContext` for workflow time, timers, conditions, Activities, signals, and queries. Do not perform database, gRPC, HTTP, email, filesystem, environment, UUID generation, ordinary `Date()` reads, or other nondeterministic side effects directly in a Workflow.

Use this shape:

```swift
@Workflow
package struct ReservationWorkflow {
    private var confirmed = false
    private var state = ReservationWorkflowState.inProgress

    package struct Input: Codable, Sendable {
        package let reservationId: UUID

        package init(reservationId: UUID) {
            self.reservationId = reservationId
        }
    }

    package mutating func run(
        context: WorkflowContext<Self>,
        input: Input
    ) async throws -> ReservationWorkflowResult {
        // Resolve state and execute side effects through Activities.
    }
}
```

Keep Workflow input nested under the Workflow. Make all values crossing Temporal boundaries `Codable` and `Sendable`. Return an `XWorkflowResult`, not a bare UUID or other scalar. Keep workflow state `.inProgress` until an Activity establishes a definitive `.completed`, `.expired`, or feature-specific terminal outcome. Do not add states that do not affect behavior.

Set terminal workflow state only after the corresponding Activity succeeds. Let cancellation and Activity failures propagate instead of reporting a terminal outcome prematurely.

## Activity design and retries

Put every external side effect in an Activity: database work, remote service calls, email delivery, and provider operations. Group feature Activities in one `@ActivityContainer` and inject one narrow Core service protocol:

```swift
@ActivityContainer
package struct ReservationActivities {
    private let service: any ReservationActivityServiceProtocol

    package init(service: any ReservationActivityServiceProtocol) {
        self.service = service
    }

    package struct ReserveInput: Codable, Sendable {
        package let reservationId: UUID
    }

    @Activity
    package func reserve(input: ReserveInput) async throws -> Reservation {
        try await service.reserve(reservationId: input.reservationId)
    }
}
```

Nest Activity input values under the container and use `Id`, not `ID`, in Swift property and parameter names. Use `creationDate`, `expiryDate`, and similar noun-based date names; do not use `createdAt`, `expiresAt`, or other `somethingAt` names.

Assume every Activity can be retried. Enforce idempotency with stable business identifiers, database constraints, conditional updates, or an external provider's idempotency key. Do not rely on in-memory flags.

Configure explicit Activity timeouts and retries. Leave dependency outages retryable. Translate invalid state, conflicting canonical data, and permanent consistency failures to non-retryable `ApplicationError` values at the Activity boundary. Do not catch `ActivityError` in Workflow code merely to inspect `ApplicationError.type` strings; model the operation so permanent failures can fail the Workflow cleanly.

## Workflow clients

Define the caller-facing client protocol in Core:

```swift
package protocol ReservationWorkflowClient: Sendable {
    func start(reservationId: UUID) async throws
    func confirm(reservationId: UUID) async throws
    func state(reservationId: UUID) async throws -> ReservationWorkflowState
}
```

Implement it in `<Service>Workflows` as `TemporalReservationWorkflowClient`. Store only the long-lived `TemporalClient` and task queue. Do not add custom `CallOptions` unless the established transport requires them.

Derive a deterministic workflow ID from the immutable domain identifier:

```swift
private static func workflowId(_ reservationId: UUID) -> String {
    "catalog-reservation-\(reservationId.uuidString.lowercased())"
}
```

Start with:

```swift
WorkflowOptions(
    id: Self.workflowId(reservationId),
    taskQueue: taskQueue,
    idReusePolicy: .rejectDuplicate,
    idConflictPolicy: .useExisting
)
```

Map Temporal failures into the Core client's small typed error contract. Do not expose Temporal SDK errors from Core or gRPC.

## Transactions and durability

Never call Temporal while a PostgreSQL transaction is open. Complete the local atomic write first, then start or signal the Workflow for low latency:

```swift
let reservation = try await database.withTransaction { context in
    // Persist the complete local business state.
}
try await workflowClient.start(reservationId: reservation.id)
```

Use Temporal history as the durable execution state. Do not add a periodic worker-side database scanner, a second workflow-state table, signal-with-start, or logic that reconstructs and resumes a Workflow from persisted business state by default. Keep start and signal as distinct operations. Add cross-system repair infrastructure only when the user explicitly accepts its additional consistency model and complexity.

## Signals, queries, and state

Use a signal for a command that informs an existing Workflow. The client method returns after Temporal accepts the signal; it must not wait for Workflow completion or fetch the Workflow result:

```swift
@WorkflowSignal
package mutating func confirmed(input: Void) {
    confirmed = true
}
```

Use a query only for observation:

```swift
@WorkflowQuery
package func currentState(input: Void) throws -> ReservationWorkflowState {
    state
}
```

Signals mutate small deterministic Workflow fields. Queries return current Workflow state without causing side effects. Keep persisted business state and `XWorkflowState` separate; give each only the cases its owner needs.

## Timers and cancellation

For an expiring condition, use `WorkflowContext.timeout` with `condition`:

```swift
let confirmed = try await context.timeout(
    for: .seconds(expiryDate.timeIntervalSince(context.now))
) {
    do {
        try await context.condition { $0.confirmed }
        return true
    } catch is CanceledError {
        return false
    }
}
```

The Swift Temporal SDK timeout races its durable sleep against the body, cancels the losing child, waits for that child, and returns or rethrows the body's result. `condition` throws Temporal's `CanceledError` when the timer cancels it. Catch `CanceledError`, not Swift's `CancellationError`; do not invent a timeout error and do not add `Task.checkCancellation()` to this pattern.

On the false branch, execute the expiration Activity before assigning `.expired`:

```swift
guard confirmed else {
    try await context.executeActivity(/* expiration Activity */)
    state = .expired
    return ReservationWorkflowResult(state: state)
}
```

This preserves cancellation: if the parent Workflow was cancelled rather than merely timed out, the subsequent Temporal operation observes cancellation and throws before terminal state is assigned.

## Worker composition

Register `Worker.self` beside `Serve.self` and `Database.self` in the executable command tree. Run `TemporalWorker` from the `worker` subcommand, not from the gRPC `serve` command.

The worker composition root owns:

- configuration and logging;
- one Postgres client when Activities need persistence;
- long-lived gRPC/provider clients used by Activities;
- the concrete Core Activity service;
- one `TemporalWorker` with explicit workflow definitions and Activity containers;
- one `ServiceGroup` containing the worker and all its long-lived dependencies.

Use the same Temporal namespace and task queue in the server's `TemporalClient`, its workflow-client adapter, and the worker. Manage the client and worker with graceful shutdown signals. Do not run a cancellation-aware reconciliation service beside the worker.

## Naming and file style

Use this feature-local layout:

```text
Sources/<Service>Workflows/<Feature>/
  <Feature>Workflow.swift
  <Feature>Activities.swift
  Temporal<Feature>WorkflowClient.swift
```

Use `package` access across targets, `private` mutable Workflow fields, nested `Input` values, and nested Activity input values. Preserve the repository's conditional Foundation imports and import ordering. Name identifiers `xId`, Workflow types `XWorkflow`, Activity containers `XActivities`, Core adapters `XWorkflowClient`, and Temporal implementations `TemporalXWorkflowClient`.

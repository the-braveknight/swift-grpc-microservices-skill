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

Put every external side effect in an Activity: database work, remote service calls, email delivery, and provider operations. Give each Activity one side effect. Minting a secret and delivering it are two: issue the challenge in one Activity, send it in the next, so a failing delivery retries against the same stored digest instead of rotating the value the recipient already holds. Deleting a secret is a third, run before terminal state is assigned.

Give an activity a distinct registration name whenever two `@ActivityContainer`s on one worker would otherwise share it. The Temporal activity name defaults to the method name, and every container registered on a worker shares one activity-name space — so a `recordPurchase` in two containers registers the same name twice and one silently overrides the other, running the wrong implementation with no error. The Swift `Activities.<Method>` type is still keyed by the Swift method, so only the registration name collides: set `@Activity(name: "RecordLicensePurchase")` on one (or both) and leave the workflow call sites unchanged. Watch the worker log for `Duplicate activity registration` — it is the symptom.

Group feature Activities in one `@ActivityContainer` and inject one narrow Core service protocol:

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

Nest Activity input values under the container and use `Id`, not `ID`, in Swift property and parameter names; strict camel case keeps `registrationId` aligned with SQL `registration_id` and proto `registration_id` without acronym special-casing. Use `creationDate`, `expirationDate`, and similar noun-based date names; do not use `createdAt`, `expiresAt`, `expiryDate`, or other `somethingAt` names.

Assume every Activity can be retried after its side effect succeeds but before Temporal receives the result. Make each write retry-safe at the system that owns the side effect: use unique constraints for creates, compare-and-swap updates for transitions, and provider idempotency keys for email, payments, or messaging. Do not rely on Workflow fields, Activity memory, or Temporal history as the downstream idempotency mechanism.

Derive a stable, namespaced idempotency key from immutable Workflow input or a caller-owned pending-record identifier, such as `authentication-registration-<registrationId>`. Reuse the exact key on every Activity attempt; never generate a UUID inside Workflow code or per Activity retry. Pass the key through the Core Activity service port and remote mutation contract.

The receiving service must atomically guarantee that the same key and canonical input returns the original result, including its database-generated entity ID, while the same key with different input produces a permanent idempotency conflict. Map that conflict to a non-retryable Activity failure. Do not treat an idempotency key as authentication or proof that a preceding workflow step occurred.

Do not generate or preallocate an entity identifier owned by another service. Pass the caller-owned workflow or pending-record identifier through the Workflow, let the owning service return its database-generated identifier during Activity execution, then persist that returned foreign identifier through retry-safe local Activity work. Require an owner-supported idempotency key for a remotely retried create. Do not convert `ALREADY_EXISTS` into success by looking up a business key such as email; distinct logical operations can carry identical business data.

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

These two policies already express the intent: a running Workflow returns the existing handle without error, and only a closed one rejects the start. Do not add a `catch is WorkflowAlreadyStartedError` on top — with a deterministic ID derived from a freshly created record it is unreachable, and where it can fire it reports success for a Workflow that will never run again.

Do not flatten every Temporal failure into a single `unavailable` case. Let the SDK error propagate from the adapter and classify it where the distinction is actionable; a one-case enum erases the cause without adding a distinction.

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

A signal-only RPC cannot return an identifier that a later Activity has not created yet. Return an empty acknowledgement from that RPC and expose the generated identifier only through the owning service or a later query/result boundary. Do not pre-generate the identifier merely to populate the signal response.

## Timers and cancellation

For an expiring condition, use `WorkflowContext.timeout` with `condition`:

```swift
let confirmed = try await context.timeout(
    for: .seconds(expirationDate.timeIntervalSince(context.now))
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

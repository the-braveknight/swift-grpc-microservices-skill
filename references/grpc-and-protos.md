# Shared protobufs and gRPC boundaries

## Contents

- Canonical proto package
- Contract design
- Producer adapter
- Consumer adapter

## Canonical proto package

Store contracts in a separate `<project>-protos` SwiftPM repository at `https://github.com/<organization>/<project>-protos.git`. Give each service its own library target/product and nest proto paths by organization, service, and API version:

```text
<project>-protos/
  Package.swift
  Sources/
    CatalogProtos/
      CatalogProtos.swift
      grpc-swift-proto-generator-config.json
      <organization>/
        catalog/
          v1/
            catalog.proto
    AccountsProtos/
      AccountsProtos.swift
      grpc-swift-proto-generator-config.json
      <organization>/
        accounts/
          v1/
            accounts.proto
```

Match the file package declaration to its path:

```proto
syntax = "proto3";

package <organization>.catalog.v1;
```

Keep `v1` even for the first release. Make compatible additions within `v1`; create `v2` for breaking API changes. Never reuse removed field numbers or rename package identity casually.

Each target uses the `GRPCProtobufGenerator` plugin with this configuration:

```json
{
  "generate": {
    "clients": true,
    "servers": true,
    "messages": true
  },
  "generatedSource": {
    "accessLevel": "public"
  }
}
```

The proto package exports a `.library(name: "<Service>Protos", targets: ["<Service>Protos"])`. Its target directly depends on `GRPCCore`, `GRPCProtobuf`, and `SwiftProtobuf`. Keep an empty marker Swift file when SwiftPM needs a Swift source beside proto resources.

Release and tag the proto package before adding a remote dependency to producer and consumer. Do not use local path dependencies in the finished integration, and do not duplicate `.proto` files in service repositories.

## Contract design

Name RPCs for business capabilities, not CRUD tables. Include only the fields consumers need.

For create operations, omit the entity identifier from the request when the service owns that entity. Let the owning database generate it and return it in the response entity. Keep identifiers as protobuf strings on the wire and parse or format UUIDs at the transport boundary. Format UUID strings lowercased on the wire in every service; Foundation's `uuidString` is uppercase, so convert with `.lowercased()` explicitly. Do not let a caller choose an owned entity identifier merely to make retries convenient; define a separate `idempotency_key` when the mutation can be retried:

```proto
message CreateItemRequest {
  string name = 1;
  string idempotency_key = 2;
}
```

Treat `idempotency_key` as required through producer validation even though proto3 strings default to empty. Name it for its transport semantics rather than leaking a caller concept such as `registration_id`. Each caller generates or derives one stable, namespaced key per logical operation and reuses it across retries. The key is not a secret, caller authentication, or permission to perform the mutation.

Where a service has two audiences — end users and the processes that act on its data — give each its own gRPC service in the same proto package (`ItemService` beside `ItemReconciliationService`), so the two contracts are separately gated and neither surface carries the other's RPCs.

Give an RPC that issues a session or a service token its own response type. A process gets no refresh token, and a session with an empty refresh field would make the contract lie.

## Producer adapter

Implement the generated `SimpleServiceProtocol` in `<Service>GRPC`. Inject use-case protocol existentials, not databases or repositories:

```swift
package struct ItemService: <Organization>_Catalog_V1_ItemService.SimpleServiceProtocol {
    private let createItemUseCase: any CreateItemUseCaseProtocol
    private let listItemsUseCase: any ListItemsUseCaseProtocol

    package init(
        createItemUseCase: any CreateItemUseCaseProtocol,
        listItemsUseCase: any ListItemsUseCaseProtocol
    ) {
        self.createItemUseCase = createItemUseCase
        self.listItemsUseCase = listItemsUseCase
    }
}
```

Each handler first makes its identity check (see *Identifying a caller versus requiring one* in [identity-and-access.md](identity-and-access.md)), then translates the request to a use-case input, calls the use case, and translates typed failures explicitly:

| Use-case failure | gRPC code |
| --- | --- |
| invalid input | `.invalidArgument` |
| duplicate/conflict | `.alreadyExists` |
| idempotency key reused with different input | `.failedPrecondition` |
| missing entity | `.notFound` |
| authentication failure | `.unauthenticated` |
| authorization failure | `.permissionDenied` |
| unexpected internal failure | `.internalError` |

Keep stable, non-sensitive messages. Do not expose database errors.

Keep the service implementation at the feature root. Put conversions in a feature-local `Protobuf/` directory:

```swift
// Sources/CatalogGRPC/Items/Protobuf/Item+Protobuf.swift
extension <Organization>_Catalog_V1_Item {
    init(item: Item) {
        self.init()
        self.id = item.id.uuidString.lowercased()
        self.name = item.name
        self.creationDate = Google_Protobuf_Timestamp(date: item.creationDate)
    }
}
```

Map each request to its use-case input in a semantic `XUseCaseInput+Protobuf.swift` file. Perform transport validation — UUID parsing, integer range checks, enum recognition, timestamp conversion — in that initializer instead of adding generic free helpers to the service implementation:

```swift
// Sources/CatalogGRPC/Items/Protobuf/CreateItemUseCaseInput+Protobuf.swift
extension CreateItemUseCaseInput {
    init(request: <Organization>_Catalog_V1_CreateItemRequest) throws {
        self.init(name: request.name, idempotencyKey: request.idempotencyKey)
    }
}
```

Refuse an enum value the service cannot interpret rather than reading it as a default: `.unspecified` is an unset field and `.UNRECOGNIZED` is a value added to the contract after this service was built, and mapping either to the least-privileged case silently changes meaning.

Do not mark a conversion extension `private` when another file in the GRPC target calls it. Use only one `Protobuf/` level per feature; do not add request/response subdivisions or generic `Mappings` and `Adapters` folders.

Expose the set of RPCs excluded from caller identification as a named `Set<MethodDescriptor>` built from the generated `Method.<Name>.descriptor` statics in the transport target, so it cannot drift from the proto and the composition root does not spell descriptors.

## Consumer adapter

Keep the consumer's caller-facing use-case protocol, input, error, and local entity. Replace only the concrete implementation:

```swift
package struct ListCatalogItemsUseCase: ListCatalogItemsUseCaseProtocol {
    private let client: <Organization>_Catalog_V1_ItemService.ClientProtocol

    package init<Transport>(client: GRPCClient<Transport>) {
        self.client = <Organization>_Catalog_V1_ItemService.Client(wrapping: client)
    }

    package func callAsFunction() async throws -> [Item] {
        let response = try await client.listItems(.init())
        return try response.items.map { message in
            guard let id = UUID(uuidString: message.id) else {
                throw ListCatalogItemsUseCaseError.malformedResponse
            }
            return Item(id: id, name: message.name, creationDate: message.creationDate.date)
        }
    }
}
```

Catch `RPCError` and map known status codes to the existing local use-case error. Do not catch old repository errors after the implementation becomes remote. Decide how unavailable/deadline failures appear to the caller; do not silently collapse all transport failures into a business conflict, and do not funnel them into a single `.unavailable` case (see *Boundary rules* in [architecture.md](architecture.md)).

Construct one long-lived `GRPCClient` in the consuming executable, wrap it in generated service clients inside the consumer implementation, inject it, and add it to `ServiceGroup`. Do not create a client per request. Register interceptors — caller forwarding, service identity — on the client, never per call.

Use deadlines and retry policy only from explicit latency and idempotency requirements. Never automatically retry a non-idempotent mutation without a contract-level idempotency design. Generate or derive the key once outside the retry loop; do not produce a new key for each attempt.

# Shared protobufs and gRPC boundaries

## Contents

- Canonical proto package
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
      example/
        catalog/
          v1/
            catalog.proto
    AccountsProtos/
      AccountsProtos.swift
      grpc-swift-proto-generator-config.json
      example/
        accounts/
          v1/
            accounts.proto
```

Replace `example` with the organization's stable reverse-domain or namespace segment. Match the file package declaration to its path:

```proto
syntax = "proto3";

package example.catalog.v1;
```

Keep `v1` even for the first release. Make compatible additions within `v1`; create `v2` for breaking API changes. Never reuse removed field numbers or rename package identity casually.

Each target uses the `GRPCProtobufGenerator` plugin and this generator configuration:

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

Release and tag the proto package before adding a remote dependency to producer and consumer. Use a semantic version requirement:

```swift
.package(
    url: "https://github.com/<organization>/<project>-protos.git",
    from: "0.1.0"
)
```

Do not use local path dependencies in the finished integration and do not duplicate `.proto` files in service repos.

## Producer adapter

Implement the generated `SimpleServiceProtocol` in `<Service>GRPC`. Inject use-case protocol existentials, not databases or repositories:

```swift
package struct ItemService: Example_Catalog_V1_ItemService.SimpleServiceProtocol {
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

Translate request messages to use-case inputs. Translate typed failures explicitly:

| Use-case failure | gRPC code |
| --- | --- |
| invalid input | `.invalidArgument` |
| duplicate/conflict | `.alreadyExists` |
| missing entity | `.notFound` |
| authentication failure | `.unauthenticated` |
| authorization failure | `.permissionDenied` |
| unexpected internal failure | `.unknown` or `.internal` according to the established API |

Keep stable, non-sensitive messages. Do not expose database errors.

Place entity/message conversion in the feature folder:

```swift
// Sources/CatalogGRPC/Items/Item+Protobuf.swift
extension Example_Catalog_V1_Item {
    init(item: Item) {
        self.init()
        self.identifier = item.identifier
        self.creationDate = Google_Protobuf_Timestamp(date: item.creationDate)
    }
}
```

Do not mark the conversion extension `private` when another file in the GRPC target calls it. Do not create `Mappings` or `Adapters` folders.

## Consumer adapter

Keep the consumer's caller-facing use-case protocol, input, error, and local entity when HTTP/API code already depends on them. Replace only its concrete implementation:

```swift
public struct ListCatalogItemsUseCase: ListCatalogItemsUseCaseProtocol {
    private let client: CatalogProtos.Example_Catalog_V1_ItemService.ClientProtocol

    public init<Transport>(client: GRPCClient<Transport>) {
        self.client = Example_Catalog_V1_ItemService.Client(wrapping: client)
    }

    public func callAsFunction() async throws -> [Item] {
        let response = try await client.listItems(.init())

        return response.items.map {
            Item(
                identifier: $0.identifier,
                creationDate: $0.creationDate.date
            )
        }
    }
}
```

Catch `RPCError` and map known status codes to the existing local use-case error. Do not catch old repository errors after the implementation becomes remote. Decide how unavailable/deadline failures appear to the caller; do not silently collapse all transport failures into a business conflict.

Construct one long-lived `GRPCClient` in the consuming executable, wrap it in generated service clients inside the consumer implementation, inject it, and add it to `ServiceGroup`. Do not create a client per request.

Use deadlines/retry policy only from explicit latency and idempotency requirements. Never automatically retry a non-idempotent mutation without a contract-level idempotency design.

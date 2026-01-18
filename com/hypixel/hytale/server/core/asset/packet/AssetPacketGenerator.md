---
description: Architectural reference for AssetPacketGenerator
---

# AssetPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.packet
**Type:** Abstract Base Class / Strategy

## Definition
```java
// Signature
public abstract class AssetPacketGenerator<K, T extends JsonAssetWithMap<K, M>, M extends AssetMap<K, T>> {
```

## Architecture & Concepts
The AssetPacketGenerator is an abstract base class that defines a strategy for serializing asset state changes into network packets. It serves as a crucial bridge between the server-side asset management system and the network protocol layer.

Its primary role is to decouple the logic of how assets are stored and managed in memory from the specific binary format required for network transmission. By using generics, this class provides a reusable template for any asset type that conforms to the JsonAssetWithMap and AssetMap contracts.

Concrete implementations of this class are responsible for handling specific asset types. For example, a BlockAssetPacketGenerator would handle serializing block data, while an ItemAssetPacketGenerator would handle item data. This pattern allows the core asset synchronization logic to remain generic and extensible, simply delegating the final serialization step to the appropriate registered generator. This component is exclusively used on the server to prepare data for clients.

### Lifecycle & Ownership
- **Creation:** This class is abstract and is never instantiated directly. Concrete subclasses are instantiated by a higher-level service, such as an AssetSynchronizationService, typically during server bootstrap. These concrete instances are then registered against the asset types they manage.
- **Scope:** An instance of a concrete generator is stateless and its lifecycle is tied to the owning service. It persists for the entire duration of the server session.
- **Destruction:** Instances are garbage collected when the server shuts down and the services that hold references to them are destroyed.

## Internal State & Concurrency
- **State:** This class is designed to be completely stateless. All methods operate exclusively on the arguments provided and do not modify any internal fields between invocations. This makes implementations highly predictable and reusable.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Methods can be invoked by multiple threads concurrently without risk of race conditions or data corruption within the generator itself.

**Warning:** While the generator is thread-safe, the caller is responsible for ensuring that the asset maps and update queries passed as arguments are accessed in a thread-safe manner. The generator performs no synchronization on the input data.

## API Surface
The public API consists of three abstract methods for generating packets corresponding to the three primary lifecycle events of an asset collection: initialization, update, and removal.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(M, Map<K, T>) | Packet | O(N) | Serializes a full collection of assets for initial client synchronization. N is the size of the asset map. |
| generateUpdatePacket(M, Map<K, T>, AssetUpdateQuery) | Packet | O(C) | Serializes a set of created or modified assets based on a query. C is the number of changes. |
| generateRemovePacket(M, Set<K>, AssetUpdateQuery) | Packet | O(R) | Serializes a set of removed asset keys. R is the number of removals. May return null if no packet is necessary. |

## Integration Patterns

### Standard Usage
The AssetPacketGenerator is not used directly by most game logic. Instead, a central asset management service invokes the appropriate generator when it detects changes that must be synchronized with clients.

```java
// A hypothetical AssetSynchronizationService using a generator
// Assume 'blockGenerator' is the registered instance for Block assets

AssetUpdateQuery changes = assetService.getPendingBlockChanges();
if (changes.hasRemovals()) {
    Packet removePacket = blockGenerator.generateRemovePacket(blockMap, changes.getRemovedKeys(), changes);
    if (removePacket != null) {
        networkManager.broadcast(removePacket);
    }
}
if (changes.hasUpdates()) {
    Packet updatePacket = blockGenerator.generateUpdatePacket(blockMap, changes.getUpdatedAssets(), changes);
    networkManager.broadcast(updatePacket);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Concrete subclasses must not store any state between method calls. Caching data or maintaining internal collections violates the core stateless design and will lead to severe concurrency issues.
- **Ignoring Null Return:** The generateRemovePacket method is nullable. Callers must check for a null return value, which indicates that no removal packet needs to be sent. Failure to do so will result in a NullPointerException.
- **Incorrect Generator Usage:** Using a generator for the wrong asset type will lead to serialization errors or client-side exceptions. The controlling service is responsible for mapping asset types to the correct generator implementation.

## Data Pipeline
The AssetPacketGenerator is a key transformation step in the server-to-client asset synchronization pipeline.

> Flow:
> Server-side Asset Modification -> AssetMap Change Detection -> AssetUpdateQuery Creation -> **AssetPacketGenerator** -> Network Packet Serialization -> Network Dispatch -> Client |


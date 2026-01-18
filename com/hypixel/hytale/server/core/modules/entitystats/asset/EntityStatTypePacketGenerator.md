---
description: Architectural reference for EntityStatTypePacketGenerator
---

# EntityStatTypePacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset
**Type:** Utility

## Definition
```java
// Signature
public class EntityStatTypePacketGenerator extends SimpleAssetPacketGenerator<String, EntityStatType, IndexedLookupTableAssetMap<String, EntityStatType>> {
```

## Architecture & Concepts
The EntityStatTypePacketGenerator is a specialized, stateless serializer that acts as a bridge between the server-side Asset Management system and the client-facing Network Protocol Layer. Its sole responsibility is to translate changes in the server's representation of EntityStatType assets into compact, network-efficient packets for client synchronization.

This class embodies the principle of *index-based asset synchronization*. To minimize network bandwidth, the server avoids sending full string identifiers (e.g., "hytale:health") for assets. Instead, it assigns a unique, stable integer index to each asset. This generator consumes an IndexedLookupTableAssetMap, which maintains this string-to-index mapping, and constructs UpdateEntityStatTypes packets that use these integer indices.

It is a concrete implementation of the generic SimpleAssetPacketGenerator, tailored specifically for handling the lifecycle (initialization, updates, removals) of EntityStatType assets.

## Lifecycle & Ownership
- **Creation:** This is a stateless utility class. A single instance is typically created by its owning service—likely an AssetTypeManager responsible for EntityStatType assets—during server bootstrap. It does not manage any resources and is cheap to instantiate.
- **Scope:** The object is designed to be a long-lived singleton within the scope of its owner. A single instance can and should be reused for the entire duration of the server's runtime.
- **Destruction:** As a stateless object with no managed resources, it requires no explicit destruction logic. It is eligible for garbage collection when its owning AssetTypeManager is shut down.

## Internal State & Concurrency
- **State:** The EntityStatTypePacketGenerator is **stateless and immutable**. It contains no member fields and its methods are pure functions whose output depends solely on their input arguments. It does not cache data or maintain any state between calls.

- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, its methods can be invoked concurrently from multiple threads without any risk of data corruption or race conditions. Callers must ensure that the provided Map and Set arguments are not mutated by other threads during a generator method's execution.

## API Surface
The public API provides methods to generate packets for the three fundamental asset synchronization operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Generates a full synchronization packet for all existing assets. Used to initialize a client's state. Throws IllegalArgumentException if a key in assets is not found in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Generates an incremental update packet for new or modified assets. Throws IllegalArgumentException if a key in loadedAssets is not found in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Generates a packet to instruct the client to remove one or more assets by their index. Throws IllegalArgumentException if a key in removed is not found in the assetMap. |

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and is not intended for direct use by general game logic. The server's asset management framework invokes it in response to asset loading, reloading, or unloading events.

```java
// Within an AssetTypeManager or similar system
IndexedLookupTableAssetMap<String, EntityStatType> statAssetMap = ...;
EntityStatTypePacketGenerator generator = new EntityStatTypePacketGenerator();

// On initial asset load for a client
Map<String, EntityStatType> allStats = statAssetMap.getAssets();
Packet initPacket = generator.generateInitPacket(statAssetMap, allStats);
networkService.sendToClient(client, initPacket);

// On a hot-reload of a single asset
Map<String, EntityStatType> updatedStat = ...;
Packet updatePacket = generator.generateUpdatePacket(statAssetMap, updatedStat);
networkService.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Do not call any generator method with an IndexedLookupTableAssetMap that is out of sync with the provided asset data. The map must contain an index for every key passed in the second argument, otherwise an unrecoverable IllegalArgumentException will be thrown.
- **Incorrect Packet Usage:** Do not send an `AddOrUpdate` packet to a client that has not first received an `Init` packet. The client's index mapping will be incomplete, leading to deserialization failures or state corruption. The asset synchronization lifecycle must be respected.

## Data Pipeline
This generator is a critical step in the server-to-client asset synchronization pipeline. It transforms high-level asset objects into a low-level network representation.

> Flow:
> Asset File Change -> Asset Hot-Reload Service -> AssetTypeManager -> **EntityStatTypePacketGenerator** -> UpdateEntityStatTypes Packet -> Server Network Layer -> Client Network Layer -> Client Asset Registry


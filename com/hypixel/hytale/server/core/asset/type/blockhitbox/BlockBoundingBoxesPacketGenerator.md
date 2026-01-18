---
description: Architectural reference for BlockBoundingBoxesPacketGenerator
---

# BlockBoundingBoxesPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.blockhitbox
**Type:** Utility

## Definition
```java
// Signature
public class BlockBoundingBoxesPacketGenerator extends SimpleAssetPacketGenerator<String, BlockBoundingBoxes, IndexedLookupTableAssetMap<String, BlockBoundingBoxes>> {
```

## Architecture & Concepts
The BlockBoundingBoxesPacketGenerator is a specialized, stateless component within the server's asset synchronization framework. Its sole responsibility is to translate server-side representations of block collision data into optimized network packets for client consumption.

This class acts as a bridge between the server's internal asset representation (a map of string-based asset names to BlockBoundingBoxes objects) and the network protocol's representation (a map of integer IDs to raw Hitbox arrays). This translation is critical for network performance, as it replaces verbose string identifiers with compact integer indices. The IndexedLookupTableAssetMap provides this crucial string-to-integer mapping.

The generator supports three distinct synchronization operations, each corresponding to a specific method and resulting in a packet with a different UpdateType:
*   **Init:** A full, destructive synchronization. Sends all known block hitboxes to the client, intended for the initial connection sequence.
*   **AddOrUpdate:** An incremental update. Sends new or modified hitboxes, allowing for efficient hot-reloading of assets without a full resync.
*   **Remove:** An incremental deletion. Informs the client to unload specific hitboxes, identified by their integer index.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's central asset management system during the server bootstrap process. It is typically registered as a specific generator for the BlockBoundingBoxes asset type.
- **Scope:** This is a stateless utility, designed to be a long-lived singleton. Its lifetime is tied directly to the server's main asset manager, persisting for the entire server session.
- **Destruction:** De-referenced and garbage collected upon server shutdown when the parent asset management services are torn down. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is completely stateless and immutable. It contains no member fields and all of its methods are pure functions, with their output depending exclusively on their input arguments.
- **Thread Safety:** Inherently thread-safe due to its stateless nature. Its methods can be invoked concurrently from multiple threads without any risk of data corruption or race conditions.

**WARNING:** While the generator itself is thread-safe, the collections passed as arguments (e.g., IndexedLookupTableAssetMap) may not be. The caller is responsible for ensuring that these data structures are not mutated by another thread during a packet generation operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified assets. Throws IllegalArgumentException if an asset key is not found in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates an incremental update packet for new or changed assets. Throws IllegalArgumentException if an asset key is not found in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates an incremental removal packet for unloaded assets. Throws IllegalArgumentException if an asset key is not found in the assetMap. |

## Integration Patterns

### Standard Usage
This generator is not intended for direct use by game logic developers. It is invoked by a higher-level asset synchronization service that orchestrates the process of detecting asset changes and broadcasting them to clients.

```java
// Example from a hypothetical AssetSynchronizationService
IndexedLookupTableAssetMap<String, BlockBoundingBoxes> assetMap = ...;
BlockBoundingBoxesPacketGenerator generator = new BlockBoundingBoxesPacketGenerator();

// On initial client join
Map<String, BlockBoundingBoxes> allAssets = assetMap.getAssets();
Packet initPacket = generator.generateInitPacket(assetMap, allAssets);
networkManager.sendPacket(client, initPacket);

// On asset hot-reload
Map<String, BlockBoundingBoxes> updatedAssets = getOnlyUpdatedAssets();
Packet updatePacket = generator.generateUpdatePacket(assetMap, updatedAssets);
networkManager.broadcastPacket(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Do not call any generation method with an asset key that is not present in the provided IndexedLookupTableAssetMap. This will cause a runtime exception and indicates a severe logic error in the calling asset manager. The asset map is the source of truth for string-to-index mapping.
- **Incorrect Packet Usage:** Do not use generateInitPacket for small, incremental updates. This is highly inefficient as it forces the client to discard all existing hitbox data and rebuild its entire cache, defeating the purpose of the incremental update system.
- **Manual Instantiation:** While possible, creating new instances of this generator on-demand is unnecessary. It should be treated as a singleton service provided by the core engine.

## Data Pipeline
This class is a transformation stage in the server-to-client asset data pipeline. It converts a high-level, name-based data structure into a low-level, index-based network message.

> Flow:
> Server Asset Change (File Load/Unload) -> Asset Manager -> **BlockBoundingBoxesPacketGenerator** -> UpdateBlockHitboxes Packet -> Server Network Layer -> Client
---


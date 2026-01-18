---
description: Architectural reference for BlockSet, a declarative configuration for grouping game blocks.
---

# BlockSet

**Package:** com.hypixel.hytale.server.core.asset.type.blockset.config
**Type:** Data Asset

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class BlockSet
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, BlockSet>>,
   NetworkSerializable<com.hypixel.hytale.protocol.BlockSet> {
```

## Architecture & Concepts

The BlockSet class is a data-driven asset that defines a logical grouping of game blocks. It is a foundational component for systems that need to operate on collections of blocks, such as world generation, AI behavior, and tool effectiveness.

**WARNING:** This class is marked as deprecated and scheduled for removal. New development should avoid dependencies on this system and seek modern alternatives.

Architecturally, a BlockSet is not a direct container of block IDs. Instead, it functions as a **declarative filter**. Each BlockSet asset, defined in a JSON file, specifies a set of inclusion and exclusion rules based on block metadata like *BlockTypes*, *BlockGroups*, *HitboxTypes*, and *Categories*. This approach allows for flexible and maintainable block groupings without hardcoding block IDs.

The system features an inheritance model via the *parent* field, enabling BlockSets to extend and override the rules of a base BlockSet. This creates a powerful hierarchy for defining broad categories that can be further refined.

A critical design pattern to understand is the separation of configuration from runtime execution.
*   **BlockSet:** Represents the static, deserialized configuration from a JSON file—the "what".
*   **BlockSetModule:** A server-side service that consumes all loaded BlockSet instances. It resolves the inheritance chains and filter rules, computing the final, optimized integer sets of block IDs for each BlockSet. This is the "how".

This separation ensures that the expensive computation of resolving the filter rules is performed once at startup, while runtime lookups are highly efficient.

### Lifecycle & Ownership
- **Creation:** BlockSet instances are created exclusively by the Hytale AssetStore framework during server bootstrap. The static `CODEC` field defines the deserialization logic, mapping JSON properties to the object's fields. Manual instantiation is strictly forbidden.
- **Scope:** Application-scoped. Once loaded from an asset file, a BlockSet instance is registered in a static, global `ASSET_STORE` and persists for the entire server session. It is treated as a read-only, global resource.
- **Destruction:** Instances are garbage collected when the server shuts down or when a hot-reload of assets is triggered, at which point the central `AssetRegistry` is cleared.

## Internal State & Concurrency
- **State:** A BlockSet instance is **effectively immutable** post-initialization. Its state, consisting of various include/exclude filter arrays, is populated once by the asset loader. There is no public API to mutate this state at runtime.
- **Thread Safety:** The class is **thread-safe for reads**. As its internal state is not modified after the initial loading phase, multiple threads can safely invoke its getter methods or the `toPacket` method without synchronization.

**WARNING:** The lazy initialization pattern in `getAssetStore` is not inherently thread-safe if multiple threads could trigger it simultaneously. However, in practice, the asset system is initialized in a single-threaded context during server startup, mitigating this risk.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, static registry for all BlockSet assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | A convenience method to access the asset map from the store. |
| toPacket() | com.hypixel.hytale.protocol.BlockSet | O(N) | Transforms the BlockSet into a network packet. N is the number of blocks in the resolved set. This is the primary method for synchronizing block groupings with the client. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is to retrieve a pre-loaded BlockSet from the global asset map and use it to acquire a processed set of block IDs or to serialize it for network transmission.

```java
// How a developer should normally use this
import com.hypixel.hytale.server.core.modules.blockset.BlockSetModule;

// Retrieve the resolved, computed set of block IDs for a given BlockSet
IndexedLookupTableAssetMap<String, BlockSet> assetMap = BlockSet.getAssetMap();
BlockSet oreBlocks = assetMap.get("hytale:ores");

if (oreBlocks != null) {
    // The toPacket method handles the lookup in BlockSetModule and serialization
    com.hypixel.hytale.protocol.BlockSet packet = oreBlocks.toPacket();
    // networkManager.send(player, packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockSet()`. Instances created this way will not be registered in the global `AssetStore` and will not be processed by the `BlockSetModule`, rendering them useless and causing potential null pointer exceptions.
- **State Mutation:** Do not attempt to modify the arrays within a BlockSet instance after it has been loaded. This violates its read-only contract and will lead to unpredictable behavior across the application.
- **New Development:** Do not use this class for new features. Its deprecated status means it will be removed in a future version, and any new code will require a rewrite.

## Data Pipeline
The flow of data from configuration file to network packet is a multi-stage process, ensuring a clear separation of concerns and high runtime performance.

> Flow:
> 1. **File System:** A `*.json` file defines the BlockSet rules.
> 2. **AssetStore:** The server's asset loader reads the JSON file.
> 3. **BlockSet.CODEC:** This static codec deserializes the JSON data into a `BlockSet` object in memory.
> 4. **BlockSetModule:** At a later stage of startup, this module iterates over all loaded `BlockSet` objects, resolves parent dependencies, and computes the final `IntSet` of block IDs for each one. The results are cached internally.
> 5. **Runtime Call:** A system calls `blockSet.toPacket()`.
> 6. **Packet Creation:** The `toPacket` method looks up its own name in the `BlockSetModule`'s cache to retrieve the pre-computed `IntSet` and serializes it into a `protocol.BlockSet` packet.
> 7. **Network Layer:** The packet is sent to the client.


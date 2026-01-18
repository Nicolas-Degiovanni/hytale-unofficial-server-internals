---
description: Architectural reference for ReverbEffectPacketGenerator
---

# ReverbEffectPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.reverbeffect
**Type:** Utility

## Definition
```java
// Signature
public class ReverbEffectPacketGenerator extends SimpleAssetPacketGenerator<String, ReverbEffect, IndexedLookupTableAssetMap<String, ReverbEffect>> {
```

## Architecture & Concepts
The ReverbEffectPacketGenerator is a specialized, stateless transformer component within the server-side asset pipeline. Its primary function is to serialize the in-memory representation of ReverbEffect assets into a compact, network-transmissible format.

This class acts as a bridge between the server's internal asset state and the client's asset registry. It does not manage assets itself; rather, it is invoked by a higher-level asset management system when changes to ReverbEffect assets occur.

A critical architectural concept employed here is the use of an IndexedLookupTableAssetMap. To minimize network bandwidth, the server and client maintain a synchronized, integer-indexed table of assets. Instead of transmitting the full string identifier for each asset (e.g., "dungeon_large_hall"), the generator translates this key into a small integer index. This class is responsible for performing this key-to-index translation during packet construction, ensuring that the resulting network packets are highly efficient.

### Lifecycle & Ownership
-   **Creation:** An instance of ReverbEffectPacketGenerator is typically created and owned by the specific asset handler responsible for the ReverbEffect asset type. It is not a globally accessible service but an implementation detail of the reverb effect system.
-   **Scope:** The object is stateless. A single instance can be held by its parent asset handler for the entire server session, or it can be instantiated transiently as needed. Its lifetime is directly tied to the service that manages ReverbEffect assets.
-   **Destruction:** The object contains no managed resources and will be garbage collected when its owning asset handler is destroyed, typically during a server shutdown or a full asset system reload.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It holds no internal fields and all its methods are pure functions, operating exclusively on the arguments provided. The output of any method is determined solely by its input.
-   **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, multiple threads can invoke its packet generation methods concurrently without any risk of race conditions or data corruption. No external locking or synchronization is required when using this class.

## API Surface
The public API is designed to handle the three fundamental asset lifecycle events: initialization, updates, and removals.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified assets. Used to initialize a client's state. Throws IllegalArgumentException if an asset key is not in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates a delta packet containing new or modified assets. Used for hot-reloading. Throws IllegalArgumentException if an asset key is not in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a delta packet instructing the client to remove specific assets, identified by their keys. Throws IllegalArgumentException if an asset key is not in the assetMap. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by general game logic developers. It is an internal component of the asset delivery system. A higher-level service, such as an AssetTypeManager, invokes it in response to asset changes.

```java
// Conceptual example within the asset system
IndexedLookupTableAssetMap<String, ReverbEffect> assetMap = getReverbAssetMap();
Map<String, ReverbEffect> newAssets = loadNewReverbEffectsFromDisk();

// The generator is retrieved or instantiated by the framework
ReverbEffectPacketGenerator generator = new ReverbEffectPacketGenerator();

// A packet is created and dispatched to the network layer
Packet updatePacket = generator.generateUpdatePacket(assetMap, newAssets);
networkService.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **State Mismatch:** Do not call any generation method with an IndexedLookupTableAssetMap that is not perfectly synchronized with the asset collections being passed. If an asset key exists in the `assets` map but not in the `assetMap`, the system will throw an `IllegalArgumentException`, halting the operation.
-   **Incorrect Packet Usage:** Do not send an `Init` packet to a client that is already synchronized. This is wasteful and may cause unintended state resets on the client. The asset system should track client state and send the correct packet type (Init, AddOrUpdate, or Remove).

## Data Pipeline
The generator's role is to act as a serializer in the server-to-client asset data flow. It transforms a high-level, server-side data structure into a low-level, optimized network packet.

> Flow:
> ReverbEffect Asset (In-Memory POJO) -> IndexedLookupTableAssetMap (for Key-to-Index lookup) -> **ReverbEffectPacketGenerator** -> UpdateReverbEffects Packet (Network DTO) -> Server Network Layer -> Client Asset Registry


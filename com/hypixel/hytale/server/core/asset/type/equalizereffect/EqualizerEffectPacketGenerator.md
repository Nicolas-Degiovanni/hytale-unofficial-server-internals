---
description: Architectural reference for EqualizerEffectPacketGenerator
---

# EqualizerEffectPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.equalizereffect
**Type:** Utility

## Definition
```java
// Signature
public class EqualizerEffectPacketGenerator extends SimpleAssetPacketGenerator<String, EqualizerEffect, IndexedLookupTableAssetMap<String, EqualizerEffect>> {
```

## Architecture & Concepts
The EqualizerEffectPacketGenerator is a specialized, stateless component within the server's asset management framework. Its sole responsibility is to act as a translation layer, converting the server's internal representation of EqualizerEffect assets into optimized, network-ready data packets for client synchronization.

This class is a critical part of the data pipeline that ensures clients have an up-to-date and consistent view of all available audio equalizer effects. It leverages an IndexedLookupTableAssetMap to transform human-readable string identifiers (e.g., *hytale:underwater_muffle*) into compact integer indices. This conversion is a key network optimization, reducing packet size and client-side lookup complexity.

The generator handles three distinct synchronization scenarios, each corresponding to a specific method:
1.  **Initial State:** Transmitting the complete set of all equalizer effects to a client upon connection.
2.  **Incremental Update:** Sending new or modified effects, typically after a hot-reload or dynamic content update.
3.  **Removal:** Notifying clients that specific effects are no longer valid and should be purged.

### Lifecycle & Ownership
-   **Creation:** Instantiated once during server bootstrap by the central asset management system. It is typically registered as the designated packet generator for the EqualizerEffect asset type.
-   **Scope:** Singleton-like in practice. A single instance persists for the entire server session.
-   **Destruction:** The object is garbage collected during server shutdown when the asset management system is torn down. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is **completely stateless**. It contains no member fields and all required data is provided as method arguments. Its behavior is purely functional, depending only on its inputs.
-   **Thread Safety:** Inherently **thread-safe**. As a stateless utility, its methods can be invoked concurrently from any thread without risk of race conditions or data corruption. No synchronization mechanisms such as locks are required.

## API Surface
The public API is designed around the three core asset synchronization events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified assets. Throws IllegalArgumentException if an asset key is not found in the index map. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates an incremental update packet for new or changed assets. Throws IllegalArgumentException if an asset key is not found in the index map. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a packet to instruct clients to remove assets. Throws IllegalArgumentException if a key to be removed is not in the index map. |

**WARNING:** The complexity of all methods is linear, O(N), where N is the size of the input collection. Generating packets for an extremely large number of assets in a single operation can impact performance.

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and should not be invoked directly by general game logic. The server's central AssetTypeManager for EqualizerEffect assets will hold a reference to this generator and call the appropriate method in response to asset lifecycle events.

```java
// Example from a hypothetical AssetTypeManager
// This code is conceptual and illustrates the integration.

// On asset reload:
Map<String, EqualizerEffect> updatedEffects = loadUpdatedEffectsFromDisk();
Packet updatePacket = equalizerEffectPacketGenerator.generateUpdatePacket(assetIndexMap, updatedEffects);
networkService.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new EqualizerEffectPacketGenerator()`. The asset framework is responsible for its creation and lifecycle. Always retrieve it from the appropriate service registry.
-   **State Inconsistency:** Calling a generator method with an asset map that is out of sync with the provided asset index map will throw an `IllegalArgumentException`. This exception indicates a critical logic error in the calling system and must not be suppressed.
-   **Inefficient Updates:** Do not call `generateInitPacket` to send a small number of updates. This is highly inefficient as it forces the client to re-process its entire collection. Use `generateUpdatePacket` for incremental changes.

## Data Pipeline
The generator functions as a specific step in the server-to-client asset synchronization pipeline.

> Flow:
> Server Event (e.g., Mod Load) -> AssetManager loads/updates EqualizerEffect data -> **EqualizerEffectPacketGenerator** transforms data into UpdateEqualizerEffects packet -> Network Layer -> Client receives packet and updates its local asset registry.


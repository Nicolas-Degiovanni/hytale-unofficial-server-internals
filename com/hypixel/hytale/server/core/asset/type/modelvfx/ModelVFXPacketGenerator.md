---
description: Architectural reference for ModelVFXPacketGenerator
---

# ModelVFXPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.modelvfx
**Type:** Utility

## Definition
```java
// Signature
public class ModelVFXPacketGenerator extends SimpleAssetPacketGenerator<String, ModelVFX, IndexedLookupTableAssetMap<String, ModelVFX>> {
```

## Architecture & Concepts

The ModelVFXPacketGenerator is a specialized, stateless component within the server's asset synchronization framework. Its sole responsibility is to translate changes in the server-side state of ModelVFX assets into compact, efficient network packets destined for game clients.

This class is a concrete implementation of the generic SimpleAssetPacketGenerator, providing the specific strategy for handling ModelVFX data. It acts as a bridge between the high-level Asset Management system and the low-level Network Protocol layer.

The core architectural principle employed is **index-based asset mapping**. Instead of transmitting full string identifiers (e.g., "hytale:vfx/fireball") over the network, this generator relies on an IndexedLookupTableAssetMap. This map pre-assigns a unique, stable integer index to each asset identifier. The generator then uses these integers in the network packets, significantly reducing packet size and improving client-side processing speed. This integer serves as the canonical ID for the asset for the duration of the client-server session.

The generator produces a single packet type, UpdateModelvfxs, but populates it differently based on the operation:
*   **Init:** A complete snapshot of all known ModelVFX assets for a newly connecting client.
*   **AddOrUpdate:** A delta containing only new or modified assets.
*   **Remove:** A list of indices for assets that have been unloaded or removed.

## Lifecycle & Ownership

*   **Creation:** Instantiated once during server bootstrap by a higher-level manager, likely the primary AssetService or AssetSynchronizationManager. Its creation is lightweight as it is stateless.
*   **Scope:** Singleton. A single instance persists for the entire lifetime of the server process. It is designed to be shared and reused for all packet generation requests concerning ModelVFX assets.
*   **Destruction:** The object is garbage collected upon server shutdown. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency

*   **State:** This class is **stateless and immutable**. It holds no internal data between method invocations. All necessary information, such as the asset map and the list of assets to process, is provided as method arguments.
*   **Thread Safety:** The class is inherently **thread-safe**. Its methods are pure functions that transform input to output. It is safe to call any generation method from multiple threads concurrently, provided the input collections themselves are not being mutated by another thread during the execution of the method. No internal locks or synchronization primitives are used or required.

## API Surface

The public contract is defined by its parent, SimpleAssetPacketGenerator. The key methods are the protected implementations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for all specified assets. Throws IllegalArgumentException if an asset key is not present in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates a delta packet for new or updated assets. Throws IllegalArgumentException if an asset key is not present in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a delta packet to instruct the client to remove assets. Throws IllegalArgumentException if an asset key is not present in the assetMap. |

*Complexity is O(N) where N is the number of assets in the input collection.*

## Integration Patterns

### Standard Usage

This class is not intended for direct use by game logic developers. It is an internal component of the asset system, invoked by a synchronization manager that tracks asset state changes.

```java
// Invoked within a hypothetical AssetSynchronizationManager
IndexedLookupTableAssetMap<String, ModelVFX> vfxAssetMap = assetService.getModelVFXMap();
Map<String, ModelVFX> newlyLoadedVFX = getNewlyLoadedAssets();

// The generator is retrieved from a central service registry
ModelVFXPacketGenerator generator = assetService.getPacketGenerator(ModelVFX.class);

// A packet is generated and dispatched to the network layer
Packet updatePacket = generator.generateUpdatePacket(vfxAssetMap, newlyLoadedVFX);
networkManager.sendPacketToAllClients(updatePacket);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not use `new ModelVFXPacketGenerator()`. The generator is part of a larger system and must be retrieved from the appropriate service registry or context to ensure it operates on the correct, authoritative asset data.
*   **State Mismatch:** Passing an IndexedLookupTableAssetMap that is out of sync with the provided asset collections is a critical error. This will cause an `IllegalArgumentException` and indicates a severe logic flaw in the calling code. The lookup map must always be the source of truth for asset indices.
*   **Incorrect Packet Usage:** Do not send an `Init` packet to a client that is already synchronized. This is wasteful and may cause unintended state resets on the client. The `Init` packet is strictly for the initial asset handshake.

## Data Pipeline

The ModelVFXPacketGenerator is a key transformation step in the server-to-client asset synchronization pipeline.

> Flow:
> Server-side Asset Change (e.g., a new VFX is loaded) -> AssetSynchronizationManager detects the change -> **ModelVFXPacketGenerator** is invoked with the new asset data -> An UpdateModelvfxs packet is created -> Network Layer serializes and sends the packet -> Client receives and decodes the packet -> Client-side AssetStore is updated.


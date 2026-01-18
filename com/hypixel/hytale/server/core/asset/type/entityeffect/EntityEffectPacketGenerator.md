---
description: Architectural reference for EntityEffectPacketGenerator
---

# EntityEffectPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect
**Type:** Utility

## Definition
```java
// Signature
public class EntityEffectPacketGenerator extends SimpleAssetPacketGenerator<String, EntityEffect, IndexedLookupTableAssetMap<String, EntityEffect>> {
```

## Architecture & Concepts
The EntityEffectPacketGenerator is a specialized component within the server's asset synchronization framework. Its sole responsibility is to translate server-side collections of EntityEffect assets into a compressed, network-transmissible format for clients.

This class acts as a bridge between the server's in-memory asset representation and the Hytale network protocol. It embodies a critical network optimization pattern: **index-based serialization**. Instead of sending full string identifiers for each asset (e.g., "hytale:fire_aura"), it uses an IndexedLookupTableAssetMap to convert these strings into compact integer indices. The client receives these indices and uses a corresponding map to resolve them back to the correct asset. This significantly reduces packet size and bandwidth consumption, especially during the initial client connection when a large number of assets are synchronized.

As a subclass of SimpleAssetPacketGenerator, it adheres to a standardized contract for handling three distinct synchronization events: initial state dump (Init), incremental updates (AddOrUpdate), and asset removal (Remove).

### Lifecycle & Ownership
- **Creation:** This class is not a long-lived service. It is instantiated on-demand by higher-level systems, typically the core AssetService or an AssetSynchronizationManager, whenever a batch of entity effect assets needs to be sent to a client.
- **Scope:** Transient and method-scoped. An instance is created, a single `generate` method is invoked, and the resulting packet is retrieved. The instance is not retained afterward.
- **Destruction:** The object is immediately eligible for garbage collection after the packet generation method completes. It holds no resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** Stateless. The EntityEffectPacketGenerator contains no member fields and does not manage any internal state. Its output is purely a function of its input arguments, making its behavior fully deterministic.
- **Thread Safety:** This class is inherently thread-safe. Because it is stateless, a single instance could theoretically be shared and used concurrently by multiple threads without any risk of data corruption or race conditions. However, its intended lifecycle is transient instantiation.

## API Surface
The public contract is designed around the three primary asset synchronization operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for a client's initial connection. Contains all specified assets. Throws IllegalArgumentException if an asset key is not present in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates an incremental update packet containing new or modified assets. Used for live asset reloading. Throws IllegalArgumentException if an asset key is not present in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a packet instructing the client to unload specific assets, identified by their keys. Throws IllegalArgumentException if a key is not present in the assetMap. |

## Integration Patterns

### Standard Usage
This generator is invoked by the server's asset management system to prepare a network packet. The caller is responsible for providing the authoritative index map and the collection of assets to be processed.

```java
// Invoked within a server-side asset synchronization task
IndexedLookupTableAssetMap<String, EntityEffect> masterAssetMap = assetService.getEntityEffectMap();
Map<String, EntityEffect> assetsForNewClient = assetService.getAllEntityEffects();

EntityEffectPacketGenerator generator = new EntityEffectPacketGenerator();
Packet initPacket = generator.generateInitPacket(masterAssetMap, assetsForNewClient);

clientConnection.send(initPacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Never invoke a generator method with an IndexedLookupTableAssetMap that is out of sync with the provided asset collection. All keys in the asset map or set must have a corresponding entry in the lookup table. Failure to ensure this consistency will raise an unrecoverable IllegalArgumentException.
- **Incorrect Packet Type:** Do not use generateInitPacket for incremental updates. Client-side handlers are designed to perform a full state reset upon receiving an Init packet, which would be inefficient and potentially buggy if used for a small update. Use generateUpdatePacket for hot-reloading or adding new assets post-initialization.

## Data Pipeline
The generator is a key transformation step in the server-to-client asset data flow. It converts a high-level, string-keyed map into a low-level, integer-keyed packet structure.

> Flow:
> Server Asset Registry -> `Map<String, EntityEffect>` -> **EntityEffectPacketGenerator** -> `UpdateEntityEffects Packet` -> Server Network Layer -> Client Asset Registry


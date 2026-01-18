---
description: Architectural reference for BlockParticleSetPacketGenerator
---

# BlockParticleSetPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.blockparticle
**Type:** Utility

## Definition
```java
// Signature
public class BlockParticleSetPacketGenerator extends DefaultAssetPacketGenerator<String, BlockParticleSet> {
```

## Architecture & Concepts
The BlockParticleSetPacketGenerator is a specialized component within the server's asset synchronization framework. Its primary function is to act as a translator, converting in-memory BlockParticleSet asset representations into their corresponding network-routable Packet objects.

This class embodies the **Strategy Pattern**. The core asset management system is agnostic to the network format of any specific asset. It delegates the responsibility of packet creation to concrete generator implementations like this one. This design decouples the high-level asset lifecycle management (loading, unloading, hot-reloading) from the low-level details of network serialization for block particles.

It exclusively handles the creation of UpdateBlockParticleSets packets, which are used to inform clients about the initial state, subsequent updates, or removal of BlockParticleSet assets.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central AssetService during the bootstrap phase. It is registered as the designated packet generator for the BlockParticleSet asset type. It is not created on-demand.
- **Scope:** Singleton-like in practice. A single instance persists for the entire server session.
- **Destruction:** The object is garbage collected during a clean server shutdown when the AssetService is decommissioned. No explicit cleanup logic is required.

## Internal State & Concurrency
- **State:** This class is **stateless**. It maintains no internal fields or caches. Its output is purely a function of its input arguments. The state of the assets themselves is owned and managed by the calling system, typically the AssetService.
- **Thread Safety:** This class is **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads without external locking. However, the caller is responsible for ensuring that the collections passed as arguments (e.g., assets, removed) are not mutated by another thread during the execution of a generator method.

## API Surface
The public contract is defined by the DefaultAssetPacketGenerator interface. These methods are invoked by the asset management system in response to asset lifecycle events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for all existing assets. Used for initial client connection. N is the total number of assets. |
| generateUpdatePacket(loadedAssets) | Packet | O(N) | Creates a delta packet for newly added or modified assets. Used for hot-reloading. N is the number of updated assets. |
| generateRemovePacket(removed) | Packet | O(N) | Creates a delta packet to instruct clients to unload specific assets. N is the number of removed asset keys. |

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and is not intended for direct use by gameplay systems. The server's AssetService invokes it automatically when a client requires an asset state update.

```java
// Conceptual usage within the AssetService
// NOTE: This is a simplified example. Do not call this class directly.

// On asset reload:
Map<String, BlockParticleSet> reloadedAssets = ...;
Packet updatePacket = blockParticleGenerator.generateUpdatePacket(reloadedAssets);
networkManager.sendPacketToAllClients(updatePacket);

// On asset removal:
Set<String> removedAssetKeys = ...;
Packet removePacket = blockParticleGenerator.generateRemovePacket(removedAssetKeys);
networkManager.sendPacketToAllClients(removePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BlockParticleSetPacketGenerator()`. The asset framework is responsible for its lifecycle. Manually creating an instance bypasses its registration with the core asset system.
- **Incorrect Packet Usage:** Do not call generateUpdatePacket for a client's initial connection. The client relies on the `UpdateType.Init` flag from generateInitPacket to correctly reset its local asset cache. Using the wrong packet type will lead to state desynchronization.
- **Stateful Extension:** Do not extend this class to add state. Its design contract is to be a stateless transformer. Introducing state will break thread-safety guarantees and is not supported by the asset management system.

## Data Pipeline
This generator is a critical step in the server-to-client asset propagation pipeline. It transforms high-level asset data into a low-level network format.

> Flow:
> Asset Hot-Reload Event -> AssetService -> **BlockParticleSetPacketGenerator** -> UpdateBlockParticleSets Packet -> Network Layer -> Client Asset Registry<ctrl63>


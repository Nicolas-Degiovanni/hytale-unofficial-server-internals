---
description: Architectural reference for FluidFXPacketGenerator
---

# FluidFXPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.fluidfx
**Type:** Utility

## Definition
```java
// Signature
public class FluidFXPacketGenerator extends SimpleAssetPacketGenerator<String, FluidFX, IndexedLookupTableAssetMap<String, FluidFX>> {
```

## Architecture & Concepts
The FluidFXPacketGenerator is a specialized, stateless component that acts as a translator between the server's internal asset representation and the network protocol. Its primary function is to serialize collections of FluidFX asset data into binary network packets for efficient synchronization with game clients.

This class is a concrete implementation of the generic SimpleAssetPacketGenerator, indicating a broader architectural pattern for handling asset updates across the network. The generator does not manage asset state itself; it is a pure transformation utility invoked by a higher-level asset management system.

A critical aspect of its design is the reliance on an IndexedLookupTableAssetMap. This data structure provides stable integer indices for string-based asset keys. By converting human-readable names (e.g., "water_flow_effect") into compact integers, the generator significantly reduces packet size and improves lookup performance on the client. This is a foundational optimization for the Hytale asset streaming system.

## Lifecycle & Ownership
- **Creation:** This is a stateless utility class. An instance is typically created and owned by a higher-level service responsible for managing a specific asset type, such as a hypothetical FluidFXAssetManager. It is not intended to be instantiated directly by general game logic.
- **Scope:** An instance can persist for the entire server session. Due to its stateless nature, a single instance can be safely reused for all FluidFX packet generation operations.
- **Destruction:** The object holds no resources and requires no special cleanup. It is eligible for standard garbage collection when its owning service is destroyed.

## Internal State & Concurrency
- **State:** **Stateless**. The FluidFXPacketGenerator contains no member fields and does not cache any data. Each method call operates exclusively on the arguments provided.
- **Thread Safety:** **Inherently thread-safe**. As a stateless utility, a single instance can be safely invoked from multiple threads concurrently without any risk of race conditions or data corruption.

    **Warning:** While the generator itself is thread-safe, the caller is responsible for ensuring that the data structures passed as arguments (e.g., the IndexedLookupTableAssetMap) are accessed in a thread-safe manner.

## API Surface
The public contract is designed around three distinct asset synchronization scenarios: initial state, incremental updates, and removals.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for new clients. Contains all specified assets. Throws IllegalArgumentException if an asset key is not in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates a delta packet containing new or modified assets. Used for hot-reloading. Throws IllegalArgumentException if an asset key is not in the assetMap. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a packet to instruct clients to unload specific assets. Throws IllegalArgumentException if a removed key is not in the assetMap. |

## Integration Patterns

### Standard Usage
This class should be used by a central asset management service to create packets in response to changes in the underlying asset data.

```java
// Within a hypothetical AssetSynchronizationService

// Dependencies
IndexedLookupTableAssetMap<String, FluidFX> fluidFXAssetMap = ...;
FluidFXPacketGenerator packetGenerator = new FluidFXPacketGenerator();

// On initial world load for a client
Map<String, FluidFX> allFluidFX = loadAllFluidFX();
Packet initPacket = packetGenerator.generateInitPacket(fluidFXAssetMap, allFluidFX);
networkLayer.sendPacket(client, initPacket);

// On a hot-reload event
Map<String, FluidFX> updatedAssets = getUpdatedFluidFX();
Packet updatePacket = packetGenerator.generateUpdatePacket(fluidFXAssetMap, updatedAssets);
networkLayer.broadcastPacket(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Never call a generator method with an asset key that does not exist in the provided IndexedLookupTableAssetMap. The asset map is the single source of truth for asset-to-index mapping, and a mismatch will result in a runtime exception.
- **Incorrect Packet Usage:** Do not send an Init packet to a client that is already synchronized. This is wasteful and may cause unintended state resets on the client. Use Update or Remove packets for incremental changes.

## Data Pipeline
The FluidFXPacketGenerator is a key component in the server-to-client asset synchronization pipeline. It transforms high-level data into a network-ready format.

> Flow:
> Server Asset Change (e.g., file modification) -> AssetManager -> **FluidFXPacketGenerator** -> UpdateFluidFX Packet -> Network Dispatcher -> Client Asset Registry


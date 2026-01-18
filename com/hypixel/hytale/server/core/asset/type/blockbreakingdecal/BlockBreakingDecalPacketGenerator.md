---
description: Architectural reference for BlockBreakingDecalPacketGenerator
---

# BlockBreakingDecalPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.blockbreakingdecal
**Type:** Utility

## Definition
```java
// Signature
public class BlockBreakingDecalPacketGenerator extends DefaultAssetPacketGenerator<String, BlockBreakingDecal> {
```

## Architecture & Concepts
The BlockBreakingDecalPacketGenerator is a specialized component within the server's Asset Management framework. It acts as a serialization bridge, translating server-side state changes of BlockBreakingDecal assets into network packets destined for clients.

Its primary responsibility is to implement the strategy for packet creation defined by its parent, DefaultAssetPacketGenerator. This class ensures that when a client connects, or when assets are updated live, the client receives the correct data to render the visual "cracking" effects on blocks as they are being damaged. It decouples the core asset management logic from the specifics of the network protocol for this particular asset type.

## Lifecycle & Ownership
- **Creation:** Instantiated by the central AssetService during server bootstrap. An instance of this generator is mapped to the BlockBreakingDecal asset type and registered within the service's internal dispatch mechanism.
- **Scope:** This object is a stateless utility. A single instance persists for the entire lifetime of the server process.
- **Destruction:** The object is eligible for garbage collection upon server shutdown when the AssetService is decommissioned. No explicit cleanup methods are required.

## Internal State & Concurrency
- **State:** This class is **stateless**. It holds no internal fields and all of its methods are pure functions that operate solely on their input arguments. The output of any method is determined entirely by its inputs.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, instances can be safely shared and invoked across multiple threads, such as the main server thread, asset loading worker threads, or network I/O threads, without requiring any synchronization or locks.

## API Surface
The public contract is defined by the DefaultAssetPacketGenerator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Generates a full synchronization packet containing all known block breaking decals. Used for initial client connection. N is the total number of decals. |
| generateUpdatePacket(loadedAssets) | Packet | O(M) | Generates a delta packet containing only new or modified decals. Used for live asset reloading. M is the number of changed assets. |
| generateRemovePacket(removed) | Packet | O(K) | **WARNING:** This method is a no-op. It is intended to generate a packet to remove decals, but the current implementation always returns null. K is the number of removed asset keys. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the server's AssetService in response to asset lifecycle events.

```java
// Conceptual example of internal AssetService usage.
// DO NOT INVOKE THIS CLASS DIRECTLY.

// On client join:
Packet initPacket = generator.generateInitPacket(masterAssetMap, allDecals);
networkManager.sendPacket(client, initPacket);

// On asset hot-reload:
Packet updatePacket = generator.generateUpdatePacket(newlyLoadedDecals);
networkManager.broadcastPacket(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockBreakingDecalPacketGenerator()`. The server's dependency injection or service locator framework is responsible for creating and managing the lifecycle of this object.
- **Reliance on Removal:** Do not design systems that depend on the dynamic removal of block breaking decals. The `generateRemovePacket` method is non-functional, and therefore, decals removed on the server will not be removed from connected clients until a full reconnect.

## Data Pipeline
This generator sits at the boundary between the server's in-memory asset representation and the network layer, performing the final serialization step before data is sent to the client.

> Flow:
> Asset File on Disk -> AssetLoader -> AssetService State -> **BlockBreakingDecalPacketGenerator** -> UpdateBlockBreakingDecals Packet -> Network Layer -> Client Asset Cache


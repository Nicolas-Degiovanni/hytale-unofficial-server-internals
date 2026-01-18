---
description: Architectural reference for RootInteractionPacketGenerator
---

# RootInteractionPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction
**Type:** Utility

## Definition
```java
// Signature
public class RootInteractionPacketGenerator extends AssetPacketGenerator<String, RootInteraction, IndexedLookupTableAssetMap<String, RootInteraction>> {
```

## Architecture & Concepts
The RootInteractionPacketGenerator is a specialized, stateless component within the server's Asset Synchronization Framework. Its sole responsibility is to translate the server-side representation of `RootInteraction` assets into a network-efficient wire format for transmission to clients.

This class acts as a bridge between the server's in-memory asset registry and the network protocol layer. It embodies a critical optimization pattern used throughout the engine: the conversion of string-based asset identifiers into integer indices. By extending the generic AssetPacketGenerator, it adheres to a standardized contract for handling the lifecycle of a specific asset type—in this case, game interactions.

The use of an IndexedLookupTableAssetMap is fundamental to its operation. This data structure maintains a consistent, two-way mapping between a human-readable string key (e.g., *hytale:player_attack*) and a compact integer index. The generator uses this map to create packets that reference interactions by their integer index, significantly reducing packet size and client-side lookup times compared to transmitting full string identifiers.

This component is invoked by a higher-level asset management service in response to three distinct events in the asset lifecycle: initial client connection, dynamic asset loading (e.g., hot-reloading content), and asset unloading.

### Lifecycle & Ownership
- **Creation:** Instantiated once during server bootstrap by the core module responsible for managing game interactions. It is registered as a dependency for the higher-level asset synchronization service.
- **Scope:** Singleton. A single instance persists for the entire lifetime of the server. As a stateless utility, there is no need for more than one instance.
- **Destruction:** The object is garbage collected during server shutdown when all references from the core modules are released. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is purely a function of its method arguments. Each method call operates independently of any previous calls.
- **Thread Safety:** The class is **unconditionally thread-safe**. Its stateless nature ensures that it can be safely invoked from multiple threads concurrently without any risk of data corruption or race conditions. No external locking or synchronization is required when calling its methods.

## API Surface
The public API is designed to handle the three primary stages of asset synchronization: initialization, incremental updates, and removal.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for new clients. Serializes all known RootInteraction assets into an UpdateRootInteractions packet with type Init. N is the total number of interactions. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(M) | Creates an incremental update packet for newly loaded or modified assets. Serializes only the changed assets. M is the number of loaded or updated interactions. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(K) | Creates a removal packet for assets that have been unloaded. K is the number of removed interactions. The packet contains placeholder objects to signal removal to the client. |

## Integration Patterns

### Standard Usage
This generator is not intended for direct use by gameplay logic. It is exclusively managed by the server's asset synchronization system. A managing service, upon detecting a change in the RootInteraction asset pool, invokes the appropriate generator method and broadcasts the resulting packet.

```java
// Example from a hypothetical AssetSynchronizationService
// This code is conceptual and illustrates the intended integration.

// On new client join:
Packet initPacket = interactionPacketGenerator.generateInitPacket(assetMap, allInteractions);
player.sendPacket(initPacket);

// On asset hot-reload:
Packet updatePacket = interactionPacketGenerator.generateUpdatePacket(assetMap, newInteractions, query);
server.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new RootInteractionPacketGenerator()`. The server's dependency management system is responsible for creating and providing the singleton instance. Direct instantiation breaks this pattern and can lead to unpredictable behavior.
- **Stateful Wrappers:** Do not wrap this class in another class that attempts to add state. Its statelessness is a core design principle for ensuring thread safety and reusability.
- **Incorrect Packet Usage:** Using generateInitPacket for a small incremental update is highly inefficient. It forces the client to re-process its entire interaction map instead of just applying a small delta. Always use the method that corresponds to the specific asset lifecycle event.

## Data Pipeline
The generator is a key transformation step in the server-to-client asset data flow. It converts a complex, server-side object graph into a compact, serializable network message.

> Flow:
> Server Asset Registry Change -> Asset Synchronization Service -> **RootInteractionPacketGenerator** -> UpdateRootInteractions Packet -> Network Layer -> Client Asset Registry


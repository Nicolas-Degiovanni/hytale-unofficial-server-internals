---
description: Architectural reference for InteractionPacketGenerator
---

# InteractionPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction
**Type:** Utility

## Definition
```java
// Signature
public class InteractionPacketGenerator extends AssetPacketGenerator<String, Interaction, IndexedLookupTableAssetMap<String, Interaction>> {
```

## Architecture & Concepts
The InteractionPacketGenerator is a specialized, stateless transformer component within the server's asset synchronization framework. Its primary function is to translate server-side Interaction asset data into optimized, client-consumable network packets.

This class acts as a bridge between the server's internal representation of game interactions (defined in configuration files and managed by an AssetManager) and the Hytale network protocol. It implements the generic AssetPacketGenerator, providing a concrete strategy for handling the lifecycle of Interaction assets—initialization, updates, and removals.

The core architectural concept this class facilitates is **network payload optimization via indexed lookups**. Instead of sending full string identifiers for each interaction (e.g., "hytale:open_door") over the network, the server uses an IndexedLookupTableAssetMap to assign a unique, compact integer ID to each interaction. This generator constructs packets that use these integer IDs, significantly reducing bandwidth consumption. The client receives the full mapping during initialization and subsequently refers to interactions by their integer ID.

## Lifecycle & Ownership
- **Creation:** An instance of InteractionPacketGenerator is typically created and managed by a higher-level service, such as an InteractionModule or a central AssetService, during server bootstrap. It is a foundational component of the asset system.
- **Scope:** As a stateless utility, a single instance is sufficient for the entire server lifecycle. It persists for the duration of the server session.
- **Destruction:** The object is garbage collected during server shutdown when the owning module is destroyed. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is purely functional. All necessary data is provided as arguments to its methods, and each method call is an independent, idempotent operation.
- **Thread Safety:** The InteractionPacketGenerator is **unconditionally thread-safe**. Its stateless nature ensures that it can be safely shared across multiple threads without locks or synchronization primitives. It is common for asset loading and packet generation to occur on background worker threads, and this class is designed to support that pattern.

## API Surface
The public API is designed to handle the three primary states of asset synchronization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for all specified interactions. Used for initial client connection. N is the number of assets. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(N) | Creates a packet to add new or update existing interactions. Used for hot-reloading assets. N is the number of loaded assets. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(N) | Creates a packet to remove interactions from the client. Used when assets are unloaded. N is the number of removed assets. |

## Integration Patterns

### Standard Usage
This generator should not be invoked directly by game logic. It is driven by the server's asset management system in response to asset lifecycle events (e.g., loading a new mod).

```java
// A hypothetical AssetService orchestrating the update
IndexedLookupTableAssetMap<String, Interaction> interactionMap = getInteractionAssetMap();
Map<String, Interaction> newInteractions = loadNewInteractionsFromDisk();

// The service updates the central lookup table first
interactionMap.addAll(newInteractions.keySet());

// Then, it uses the generator to create the network packet
InteractionPacketGenerator generator = getInteractionPacketGenerator();
Packet updatePacket = generator.generateUpdatePacket(interactionMap, newInteractions, query);

// Finally, the packet is broadcast to relevant clients
networkSystem.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InteractionPacketGenerator()`. The server maintains a single, shared instance. Always retrieve it from the appropriate service registry or context.
- **State Desynchronization:** Never call a generator method without first updating the corresponding IndexedLookupTableAssetMap. Doing so will cause the generator to use stale or incorrect integer indices, leading to a fatal desynchronization between the client and server state.
- **Incorrect Packet Usage:** Do not use `generateInitPacket` for incremental updates. The client-side asset manager treats an `Init` packet as a command to wipe its existing state, which is inefficient and can cause visual glitches. Use `generateUpdatePacket` or `generateRemovePacket` for all post-login changes.

## Data Pipeline
The InteractionPacketGenerator sits at the end of the server-side asset processing pipeline, acting as the final step before data is serialized for the network.

> Flow:
> Asset File Change (JSON/Config) -> AssetManager Reload -> **InteractionPacketGenerator** -> UpdateInteractions Packet -> Network Layer -> Client

---


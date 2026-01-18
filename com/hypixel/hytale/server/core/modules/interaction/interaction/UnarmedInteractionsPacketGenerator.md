---
description: Architectural reference for UnarmedInteractionsPacketGenerator
---

# UnarmedInteractionsPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction
**Type:** Transient Service

## Definition
```java
// Signature
public class UnarmedInteractionsPacketGenerator extends DefaultAssetPacketGenerator<String, UnarmedInteractions> {
```

## Architecture & Concepts
The UnarmedInteractionsPacketGenerator is a specialized component within the server's asset synchronization framework. Its sole responsibility is to translate the server-side representation of default, item-less player interactions into a compact, network-efficient packet format for clients.

This class acts as a bridge between the server's internal asset model, specifically the UnarmedInteractions asset, and the network protocol layer. It implements the strategy defined by its parent, DefaultAssetPacketGenerator, to handle the lifecycle events of these specific assets: initial client connection, live updates, and removal.

A critical design assumption is hardcoded into this generator: it operates exclusively on a single, well-known asset named **Empty**. This asset is expected to contain the definitive map of all possible interactions a player can perform when no item is equipped. The generator resolves the string-based interaction names from this asset into integer indices by looking them up in the global RootInteraction asset map. This index-based serialization significantly reduces network payload size.

## Lifecycle & Ownership
- **Creation:** This class is not intended for direct instantiation. It is constructed and registered by the server's central AssetService during the server bootstrap sequence. The framework maintains a collection of such generators, one for each asset type that requires network synchronization.
- **Scope:** An instance of this generator persists for the entire lifetime of the server process. It is a stateless service, so a single instance is sufficient.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown when the parent AssetService is torn down and its references are cleared.

## Internal State & Concurrency
- **State:** The UnarmedInteractionsPacketGenerator is **stateless**. It contains no instance fields and does not cache any data between method invocations. All necessary information is provided as method arguments or accessed via a static, globally-available asset registry, RootInteraction.
- **Thread Safety:** This class is **conditionally thread-safe**. Its methods are re-entrant and do not modify any shared state. Concurrency safety is dependent on the thread safety of the static RootInteraction.getAssetMap(), which is assumed to be safe for concurrent read operations. It is safe to invoke its methods from any thread, provided the input collections are not concurrently modified.

## API Surface
The public API is an implementation of the DefaultAssetPacketGenerator contract. These methods should not be called directly by game logic developers; they are invoked by the asset synchronization framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates an UpdateUnarmedInteractions packet for initial client synchronization. N is the number of interactions defined in the "Empty" asset. |
| generateUpdatePacket(loadedAssets) | Packet | O(N) | Creates an UpdateUnarmedInteractions packet for hot-reloading or live updates. N is the number of interactions defined in the "Empty" asset. |
| generateRemovePacket(removed) | Packet | O(1) | Creates a simple Remove packet. **Warning:** This implementation ignores the input set and signals a generic removal. |

## Integration Patterns

### Standard Usage
This component is used exclusively by the server's asset management system to automate the synchronization of unarmed interaction data. A developer will never interact with this class directly. The system is triggered by asset loading or reloading events.

```java
// Conceptual example of framework usage
// This code does not exist literally, but illustrates the pattern.

AssetSynchronizationManager assetSyncManager;
DefaultAssetMap<String, UnarmedInteractions> unarmedInteractionsMap;

// On server start or asset reload, the manager invokes the generator
Packet initPacket = assetSyncManager.getGeneratorFor(UnarmedInteractions.class)
                                    .generateInitPacket(unarmedInteractionsMap, unarmedInteractionsMap.getAssets());

networkService.broadcastToClients(initPacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new UnarmedInteractionsPacketGenerator()`. The asset framework is responsible for its lifecycle.
- **Direct Invocation:** Do not call the generate methods from game logic. To update unarmed interactions, you must modify the underlying "Empty" asset file and trigger a server asset reload. The framework will then invoke this generator automatically.
- **Ignoring the "Empty" Asset Convention:** The entire logic of this class is predicated on the existence of an asset named "Empty". Attempting to sync any other UnarmedInteractions asset will fail or be ignored.
- **Expecting Granular Removal:** The `generateRemovePacket` method does not support the removal of individual interactions. It sends a blanket `Remove` command, which the client must interpret as a full clear of all unarmed interactions.

## Data Pipeline
This generator is a single, critical step in the asset-to-network data pipeline. It transforms a high-level, server-side data structure into a low-level, optimized network packet.

> Flow:
> Server Asset Loader (reads `unarmed_interactions/Empty.json`) -> UnarmedInteractions Data Model -> Asset Synchronization Event -> **UnarmedInteractionsPacketGenerator** -> UpdateUnarmedInteractions Packet -> Server Network Layer -> Client Network Layer -> Client Interaction System


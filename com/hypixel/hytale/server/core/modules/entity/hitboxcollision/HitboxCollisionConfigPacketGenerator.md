---
description: Architectural reference for HitboxCollisionConfigPacketGenerator
---

# HitboxCollisionConfigPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.entity.hitboxcollision
**Type:** Utility

## Definition
```java
// Signature
public class HitboxCollisionConfigPacketGenerator
   extends AssetPacketGenerator<String, HitboxCollisionConfig, IndexedLookupTableAssetMap<String, HitboxCollisionConfig>> {
```

## Architecture & Concepts
The HitboxCollisionConfigPacketGenerator is a specialized, stateless component within the server's asset synchronization framework. Its sole responsibility is to act as a translator between the server's internal representation of hitbox configuration assets and the binary network protocol. It serializes server-side HitboxCollisionConfig objects into discrete network packets for transmission to clients.

This class is a concrete implementation of the generic AssetPacketGenerator, tailored specifically for assets managed by an IndexedLookupTableAssetMap. This is a critical architectural detail: the generator does not just serialize data, it also transforms the string-based asset keys (e.g., "hytale:player_default") into more efficient integer indices for network transport. This integer index is the primary key used by the client to identify and update the corresponding asset.

The generator supports the three fundamental asset lifecycle events:
1.  **Initialization (Init):** A full synchronization of all known hitbox configurations, sent when a client first connects or requires a complete state reset.
2.  **Update (AddOrUpdate):** An incremental update containing only new or modified hitbox configurations.
3.  **Removal (Remove):** An instruction to the client to unload or delete specific hitbox configurations.

## Lifecycle & Ownership
-   **Creation:** This generator is instantiated by the server's central asset management system during the server bootstrap phase. It is typically registered with a higher-level service responsible for managing the lifecycle of all HitboxCollisionConfig assets.
-   **Scope:** As a stateless utility, a single instance of this class persists for the entire lifetime of the server. Its lifecycle is directly bound to the parent asset management service that owns it.
-   **Destruction:** The object is eligible for garbage collection only upon server shutdown when the asset management system is dismantled. No explicit cleanup methods are required.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no member variables and does not cache any data between method invocations. All operations are performed exclusively on the arguments provided to its methods.
-   **Thread Safety:** The component is inherently **thread-safe**. Due to its stateless nature, multiple threads can invoke its generation methods concurrently without risk of data corruption or race conditions. No external locking or synchronization is necessary when using this class.

## API Surface
The public contract is defined by its parent, AssetPacketGenerator, and provides methods to handle the full asset synchronization lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified assets. N is the number of assets. Used for initial client connection. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(N) | Creates an incremental update packet for new or modified assets. N is the number of loaded assets. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(N) | Creates a packet to instruct the client to remove assets. N is the number of removed asset keys. |

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and is not intended for direct use by general game logic systems. The server's AssetService invokes it automatically when changes to hitbox configurations are detected.

```java
// Hypothetical usage within a higher-level AssetService
IndexedLookupTableAssetMap<String, HitboxCollisionConfig> hitboxAssetMap = ...;
HitboxCollisionConfigPacketGenerator generator = new HitboxCollisionConfigPacketGenerator();

// On initial client join
Map<String, HitboxCollisionConfig> allConfigs = hitboxAssetMap.getAssets();
Packet initPacket = generator.generateInitPacket(hitboxAssetMap, allConfigs);
networkLayer.sendToClient(client, initPacket);

// On a hot-reload of an asset
Map<String, HitboxCollisionConfig> updatedConfigs = getJustUpdatedConfigs();
Packet updatePacket = generator.generateUpdatePacket(hitboxAssetMap, updatedConfigs, query);
networkLayer.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create instances of this class within game logic. It should be managed and provided by the core asset management system to ensure it operates on the correct, authoritative asset state.
-   **Incorrect Packet Type:** Do not call generateInitPacket for a small, incremental update. This is highly inefficient and sends redundant data over the network. The choice of generator method must match the type of asset change (initial load, update, or removal).
-   **Stateful Extension:** Do not extend this class to add state. Its reliability and thread-safety are derived from its stateless design.

## Data Pipeline
This generator is a key stage in the server-to-client asset synchronization pipeline. It transforms high-level asset objects into low-level network packets.

> Flow:
> Server Asset Store (File/DB) -> AssetManager loads HitboxCollisionConfig -> Asset change is detected -> **HitboxCollisionConfigPacketGenerator** serializes config and key-to-index mapping -> UpdateHitboxCollisionConfig Packet -> Network Layer -> Client receives packet -> Client Asset Store is updated.


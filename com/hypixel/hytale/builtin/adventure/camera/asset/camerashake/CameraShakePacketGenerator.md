---
description: Architectural reference for CameraShakePacketGenerator
---

# CameraShakePacketGenerator

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.camerashake
**Type:** Utility

## Definition
```java
// Signature
public class CameraShakePacketGenerator extends SimpleAssetPacketGenerator<String, CameraShake, IndexedAssetMap<String, CameraShake>> {
```

## Architecture & Concepts
The CameraShakePacketGenerator is a specialized, stateless component that acts as a translator between the server-side Asset System and the client-side game state. Its primary function is to convert high-level changes in CameraShake assets—such as additions, updates, or removals—into low-level, optimized network packets.

This class implements the strategy defined by its parent, SimpleAssetPacketGenerator. It is specifically tailored for CameraShake assets, which are identified by a String key. A core architectural feature is its reliance on an IndexedAssetMap. This data structure is critical for network performance, as it allows the generator to replace bulky String identifiers with compact integer indices in the outgoing packets. This significantly reduces bandwidth consumption when synchronizing asset state with clients.

The generator is responsible for creating three distinct types of packets, corresponding to the lifecycle of an asset:
1.  **Init:** A complete snapshot of all CameraShake assets, sent to a client upon initial connection.
2.  **AddOrUpdate:** A delta packet containing only new or modified assets.
3.  **Remove:** A delta packet containing only the indices of assets that have been removed.

This component is a foundational piece of the server's asset hot-reloading and dynamic content delivery system.

### Lifecycle & Ownership
-   **Creation:** An instance of CameraShakePacketGenerator is typically created once during server bootstrap. It is instantiated by a higher-level asset management service, such as an AssetTypeHandler, which is responsible for managing the entire lifecycle of CameraShake assets.
-   **Scope:** As a stateless utility, a single instance persists for the entire lifetime of the server. It holds no internal state and is designed to be shared.
-   **Destruction:** The object is eligible for garbage collection when its owning asset management service is shut down, which typically occurs during a graceful server shutdown.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable**. This class contains no member fields and does not cache or retain any data between method invocations. All operations are pure functions of their input arguments.
-   **Thread Safety:** **Fully Thread-Safe**. Due to its stateless nature, a single instance of CameraShakePacketGenerator can be safely invoked by multiple threads concurrently without any need for external locking or synchronization. This is crucial in a multi-threaded server environment where asset updates may be triggered by different systems.

## API Surface
The public contract is inherited and implemented from SimpleAssetPacketGenerator. These methods are not intended for direct use by gameplay logic but are invoked by the asset management framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for all known assets. N is the total number of CameraShake assets. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(K) | Creates a delta packet for new or modified assets. K is the number of assets being updated. |
| generateRemovePacket(assetMap, removed) | Packet | O(R) | Creates a delta packet to signal the removal of assets. R is the number of assets being removed. |

## Integration Patterns

### Standard Usage
This class is a system-level component and should not be invoked directly. It is designed to be provided as a strategy to a generic asset handler that manages the synchronization logic. The framework is responsible for calling the correct generator method based on the context of the asset change.

```java
// Conceptual usage within the asset management system
// This code would exist within a higher-level service.

IndexedAssetMap<String, CameraShake> assetMap = getCameraShakeAssetMap();
CameraShakePacketGenerator generator = new CameraShakePacketGenerator();

// On new client join:
Map<String, CameraShake> allAssets = assetMap.getAssets();
Packet initPacket = generator.generateInitPacket(assetMap, allAssets);
client.send(initPacket);

// On asset hot-reload:
Map<String, CameraShake> updatedAssets = getOnlyUpdatedAssets();
Packet updatePacket = generator.generateUpdatePacket(assetMap, updatedAssets);
server.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation for One-Off Use:** Do not create a new CameraShakePacketGenerator for every operation. It is stateless and designed to be a long-lived, shared instance.
-   **Incorrect Packet Generation:** Do not call generateInitPacket for a minor update. The asset framework is responsible for tracking state and determining whether a full `Init` or a partial `AddOrUpdate`/`Remove` packet is required. Using the wrong type can lead to client state desynchronization or unnecessary network traffic.

## Data Pipeline
The CameraShakePacketGenerator is a critical step in the asset synchronization pipeline, transforming server-side data into a network-ready format.

> Flow:
> Server Asset Change (e.g., file modification) -> AssetManager detects change -> **CameraShakePacketGenerator** translates `Map<String, CameraShake>` to `UpdateCameraShake` packet with integer indices -> Network Layer sends CachedPacket -> Client receives packet -> Client AssetStore updates its local `IndexedAssetMap<CameraShake>` -> Game systems can now use the new or updated CameraShake effect.


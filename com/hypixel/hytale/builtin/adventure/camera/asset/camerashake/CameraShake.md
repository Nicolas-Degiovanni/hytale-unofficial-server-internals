---
description: Architectural reference for the CameraShake asset data model.
---

# CameraShake

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.camerashake
**Type:** Asset Data Model

## Definition
```java
// Signature
public class CameraShake implements NetworkSerializable<com.hypixel.hytale.protocol.CameraShake>, JsonAssetWithMap<String, IndexedAssetMap<String, CameraShake>> {
```

## Architecture & Concepts

The CameraShake class is a passive data model that defines the properties of a camera shake effect. It is a fundamental component of Hytale's data-driven asset system, representing a specific, named configuration that can be loaded from JSON and referenced throughout the game engine.

This class does not contain any active logic for applying the shake effect. Instead, it serves as a blueprint. Game logic on the server (e.g., an explosion system) will reference a CameraShake asset by its unique ID, serialize it to a network packet using the `toPacket` method, and transmit it to the client. The client's camera rendering system then receives this packet and executes the actual camera motion based on the contained configuration.

The static `CODEC` field is the most critical component, defining the schema for how CameraShake assets are deserialized from JSON files. This allows designers and developers to define new camera shake effects without recompiling the engine, simply by adding new JSON asset files.

## Lifecycle & Ownership

-   **Creation:** CameraShake instances are not created directly via their constructor. They are instantiated exclusively by the Hytale **AssetStore** during the engine's asset loading phase. The static `CODEC` is used to parse a corresponding JSON file and populate the instance fields.
-   **Scope:** Application-scoped. Once an asset is loaded into the `AssetStore`, it persists in memory for the entire duration of the server or client session. These objects are treated as globally accessible, read-only resources.
-   **Destruction:** Instances are destroyed when the global `AssetRegistry` is shut down, which typically occurs only when the application exits. There is no per-instance garbage collection or manual destruction.

## Internal State & Concurrency

-   **State:** Effectively immutable after initial loading. All fields are populated by the `CODEC` during deserialization and are not designed to be modified at runtime. This design ensures that asset data remains consistent and predictable across all systems that reference it.
-   **Thread Safety:** The class is inherently thread-safe for read operations due to its post-load immutability. Any system can safely access a loaded CameraShake instance from any thread.

    **Warning:** The static `getAssetStore` method contains a lazy initialization check. While the engine's bootstrap process is typically single-threaded, calling this method from multiple threads *before* the AssetRegistry is fully initialized could lead to a race condition. All asset access must occur after the engine's bootstrap sequence is complete.

## API Surface

The public API is primarily static and focused on retrieving assets from the global store.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton asset store for all CameraShake assets. |
| getAssetMap() | static IndexedAssetMap | O(1) | A convenience method to get the underlying map of all loaded CameraShake assets, keyed by their string ID. |
| toPacket() | com.hypixel.hytale.protocol.CameraShake | O(1) | Serializes the asset's configuration into a network-ready packet for transmission to the client. |
| getId() | String | O(1) | Returns the unique asset identifier (e.g., "generic_explosion_shake"). |

## Integration Patterns

### Standard Usage

The correct pattern is to retrieve a pre-loaded asset from the global `AssetStore` or `AssetMap` using its known ID. Never instantiate it directly.

```java
// A game system needs to trigger a camera shake defined in assets.
// 1. Get the map of all loaded camera shake assets.
IndexedAssetMap<String, CameraShake> shakeAssets = CameraShake.getAssetMap();

// 2. Look up the desired shake by its unique ID.
CameraShake explosionShake = shakeAssets.get("hytale:explosion_heavy");

// 3. If found, serialize it to a packet to be sent to the client.
if (explosionShake != null) {
    com.hypixel.hytale.protocol.CameraShake packet = explosionShake.toPacket();
    // networkSystem.sendToClient(player, packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new CameraShake()`. This creates an orphan object that is not registered in the asset system and lacks any configured data. It will cause NullPointerExceptions if used.
-   **Runtime Modification:** Do not attempt to modify the fields of a CameraShake instance after it has been loaded. The asset system treats these objects as read-only.
-   **Premature Access:** Do not call `getAssetStore` or `getAssetMap` before the main engine asset loading phase is complete. This will result in an uninitialized or incomplete asset collection.

## Data Pipeline

The CameraShake class acts as a data container at two key stages: asset loading and network replication.

> **Asset Loading Flow:**
> JSON File on Disk (`.../camerashake/explosion.json`) -> Engine Asset Loader -> **CameraShake.CODEC** -> In-Memory `AssetStore<CameraShake>`

> **Gameplay Usage Flow:**
> Server-Side Game Event -> Game Logic retrieves `CameraShake` from AssetStore -> **CameraShake.toPacket()** -> Network Layer -> Client Receives Packet -> Client Camera System Executes Effect


---
description: Architectural reference for FluidFX
---

# FluidFX

**Package:** com.hypixel.hytale.server.core.asset.type.fluidfx.config
**Type:** Asset / Data Transfer Object

## Definition
```java
// Signature
public class FluidFX implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, FluidFX>>, NetworkSerializable<com.hypixel.hytale.protocol.FluidFX> {
```

## Architecture & Concepts
The FluidFX class is a server-side asset definition that encapsulates all visual and physical properties of an in-game fluid, such as water or lava. It serves as a bridge between static game configuration defined in JSON files and the dynamic data sent to clients for rendering.

Its primary role is to be a data container, loaded and managed by the Hytale **AssetStore** framework. The static **CODEC** field is the cornerstone of this system. It is a sophisticated deserializer that not only maps JSON fields to Java object properties but also implements an inheritance model. This allows a specific fluid definition (e.g., *swamp_water*) to inherit base properties from another (e.g., *base_water*) and override only what is necessary, promoting configuration reuse and consistency.

The class implements two key interfaces:
1.  **JsonAssetWithMap**: This marks FluidFX as a manageable asset within the global AssetRegistry, loaded from JSON and stored in a lookup map, typically indexed by a namespaced ID like *hytale:water*.
2.  **NetworkSerializable**: This signifies that a FluidFX instance can be converted into a network-ready protocol packet. This is fundamental to its purpose: defining fluid effects on the server and then serializing that definition for transmission to game clients, which then use the data for rendering.

## Lifecycle & Ownership
-   **Creation:** FluidFX instances are not created directly using the constructor. They are instantiated and populated exclusively by the **AssetStore** during the server's bootstrap phase. The static CODEC is used to parse corresponding JSON asset files (e.g., `water.json`) and construct the objects.
-   **Scope:** Asset definitions loaded into the static **ASSET_STORE** are application-scoped. They persist in memory for the entire lifetime of the server process. They are treated as canonical, shared, read-only configurations.
-   **Destruction:** The entire collection of FluidFX assets is destroyed when the server shuts down and the AssetRegistry is cleared. The Java Virtual Machine's garbage collector reclaims the memory.

## Internal State & Concurrency
-   **State:** An instance of FluidFX is effectively immutable after its initial creation by the AssetStore. Its fields represent static configuration and are not designed to be modified at runtime. However, it contains one piece of mutable state: a **SoftReference** named **cachedPacket**. This is a performance optimization to cache the serialized network packet, avoiding redundant conversion work.

-   **Thread Safety:** This class is **not** thread-safe.
    -   The lazy initialization within the static **getAssetStore** method is not synchronized. Concurrent calls during server startup could lead to race conditions. This system assumes single-threaded initialization.
    -   The **toPacket** method's caching mechanism is not atomic. If multiple threads call it on the same instance simultaneously, they may perform redundant work creating a packet, though the final state will be consistent. For its intended use (read-only access from the main game loop), this is generally safe.

    **WARNING:** Do not access the static AssetStore from multiple threads during the server's initial loading phase. Do not modify FluidFX instances after they have been retrieved from the AssetStore.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, lazily-initialized registry for all FluidFX assets. **Warning:** Not thread-safe on first call. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Convenience method to get the underlying map of all loaded FluidFX assets from the store. |
| toPacket() | com.hypixel.hytale.protocol.FluidFX | O(1) cached | Converts the asset into its network-serializable representation. The result is cached in a SoftReference for performance. |
| getId() | String | O(1) | Returns the unique namespaced identifier for this asset (e.g., "hytale:water"). |
| getUnknownFor(id) | static FluidFX | O(1) | Factory method to create a placeholder FluidFX object for an ID that could not be found in the AssetStore. |

## Integration Patterns

### Standard Usage
The correct way to use FluidFX is to retrieve a pre-loaded, canonical instance from the global asset map. Never instantiate it yourself.

```java
// Retrieve the central map of all fluid effect definitions
IndexedLookupTableAssetMap<String, FluidFX> fluidEffects = FluidFX.getAssetMap();

// Get a specific fluid effect by its namespaced ID
FluidFX waterEffect = fluidEffects.get("hytale:water");

if (waterEffect != null) {
    // Convert the definition to a network packet to be sent to a client
    com.hypixel.hytale.protocol.FluidFX packet = waterEffect.toPacket();
    
    // networkManager.sendPacket(client, packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new FluidFX()`. This bypasses the asset loading system, the CODEC, and the inheritance model. The resulting object will be uninitialized, invalid, and disconnected from the game's configuration.
-   **Runtime Modification:** Do not modify the state of a FluidFX object after retrieving it from the AssetStore. These are shared, canonical instances. Modifying one will affect all parts of the server that reference it, leading to unpredictable behavior.
-   **Concurrent Initialization:** Do not attempt to access `FluidFX.getAssetStore()` from multiple threads before the server has completed its main bootstrap sequence. The lazy initialization pattern used is not safe for concurrent access.

## Data Pipeline
The flow of FluidFX data begins as a static file and ends as a packet sent to the client for rendering.

> Flow:
> JSON File (`assets/namespace/fluids/water.json`) -> Server Bootstrap -> AssetStore Loader -> **FluidFX.CODEC Deserialization** -> In-Memory **FluidFX** Instance -> `toPacket()` -> Network Layer -> Client Renderer


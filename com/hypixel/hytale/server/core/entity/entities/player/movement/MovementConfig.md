---
description: Architectural reference for MovementConfig
---

# MovementConfig

**Package:** com.hypixel.hytale.server.core.entity.entities.player.movement
**Type:** Data Transfer Object / Asset

## Definition
```java
// Signature
public class MovementConfig implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, MovementConfig>>, NetworkSerializable<MovementSettings> {
```

## Architecture & Concepts

The MovementConfig class is a data-driven asset that defines the complete set of physical parameters governing player movement. It is not a service or a manager, but rather an immutable data container that represents a specific movement profile, such as "default", "swimming", or "flying".

Its primary role is to bridge the gap between human-readable configuration files (JSON) and the server's internal physics and networking systems. This is achieved through three core architectural features:

1.  **Asset Deserialization:** The static `CODEC` field is a critical component that dictates how the engine's `AssetStore` parses JSON files into MovementConfig objects. The extensive use of `appendInherited` within the codec's construction reveals a powerful feature of the asset system: movement configurations can form a hierarchy. A specific profile can inherit from a base configuration and override only the necessary parameters, reducing data duplication and simplifying management.

2.  **Global Asset Registry:** MovementConfig objects are managed by the global `AssetStore`. The static `getAssetStore` method provides a singleton-like access point to a central, in-memory cache of all loaded movement profiles. This ensures that the expensive process of file I/O and deserialization happens only once at startup.

3.  **Network Synchronization:** By implementing `NetworkSerializable<MovementSettings>`, this class guarantees that server-side movement physics can be perfectly replicated on the client. The `toPacket` method transforms the server-side configuration object into a `MovementSettings` network packet, which is then sent to the client. This is the authoritative mechanism for synchronizing physics constants between server and client.

## Lifecycle & Ownership

-   **Creation:** Instances are not intended to be created manually via the `new` keyword. They are instantiated by the `AssetStore` during the server's asset loading phase. The `AssetBuilderCodec` uses reflection and the defined builder logic to populate an object's fields from a corresponding JSON asset file on disk. A static `DEFAULT_MOVEMENT` instance also exists as a hardcoded fallback.
-   **Scope:** Once loaded, a MovementConfig instance is cached in the static `ASSET_STORE` for the entire lifetime of the server process. These objects should be treated as globally accessible, immutable singletons.
-   **Destruction:** Instances are cleared from memory only when the `AssetRegistry` is shut down, which typically coincides with server termination.

## Internal State & Concurrency

-   **State:** A MovementConfig object is effectively **immutable**. After being populated by the asset loader, its state—a collection of primitive float and boolean values—does not change. The public API consists entirely of getters, enforcing this read-only contract.
-   **Thread Safety:** The class is inherently **thread-safe for reads** due to its immutability. Any system can safely reference and read from a MovementConfig object from any thread without synchronization.

    **Warning:** The lazy initialization within the static `getAssetStore` method is not protected by a lock. It relies on the assumption that the `AssetRegistry` and its associated stores are fully initialized in a single-threaded context during server bootstrap. Accessing this method from multiple threads before initialization is complete would be unsafe.

## API Surface

The public API is dominated by simple getters. The architecturally significant symbols are those that manage its integration with the asset and network systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton asset store for all MovementConfig instances. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the underlying map of all loaded MovementConfig assets, keyed by their string ID. |
| toPacket() | MovementSettings | O(1) | Serializes the object's state into a network-ready `MovementSettings` packet. |
| getId() | String | O(1) | Returns the unique asset identifier for this configuration (e.g., "BuiltinDefault"). |

## Integration Patterns

### Standard Usage

A system, such as a player physics controller, retrieves a specific configuration from the global asset store and uses its values. The configuration is typically fetched once and cached within the controller.

```java
// Retrieve the default movement profile from the global asset store
IndexedLookupTableAssetMap<String, MovementConfig> assetMap = MovementConfig.getAssetMap();
MovementConfig defaultConfig = assetMap.get("BuiltinDefault");

// Apply a physics parameter during a game tick
float jumpForce = defaultConfig.getJumpForce();
player.applyVerticalImpulse(jumpForce);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new MovementConfig()`. This creates an empty, unconfigured object that bypasses the asset system entirely. All configurations must be loaded from asset files via the `AssetStore`.
-   **Per-Entity Copies:** Do not create a new copy of a MovementConfig for every player entity. These objects are immutable and designed to be shared. Retrieve the reference from the `AssetStore` and share it among all entities that use the same profile.
-   **Early Access:** Do not call `MovementConfig.getAssetStore()` before the main `AssetRegistry` has completed its initialization phase during server startup.

## Data Pipeline

The MovementConfig class is a key component in the pipeline that transforms configuration on disk into synchronized physics behavior in the game.

> Flow:
> JSON Asset File -> AssetStore Deserialization (via CODEC) -> **MovementConfig Instance** -> Player Physics System -> toPacket() -> MovementSettings Packet -> Network Layer -> Client Simulation


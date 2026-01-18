---
description: Architectural reference for WorldMapConfig
---

# WorldMapConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Data Object

## Definition
```java
// Signature
public class WorldMapConfig {
```

## Architecture & Concepts
The WorldMapConfig class is a data-driven configuration object that defines the display properties and features of the in-game world map. It is not a service or manager, but rather a passive data structure whose state is determined entirely by an external asset definition.

Architecturally, this class serves as a strongly-typed representation of a configuration file on disk. Its primary design feature is the static **CODEC** field, which integrates it directly into Hytale's serialization and asset loading pipeline. This allows game designers and modders to define world map behavior in a declarative way (e.g., using JSON or HOCON) without modifying engine code. The server loads this configuration, and it is likely synchronized to clients to ensure the UI reflects the server's intended settings.

## Lifecycle & Ownership
- **Creation:** An instance of WorldMapConfig is never created directly via its constructor in standard game logic. It is instantiated exclusively by the Hytale **Codec** system when a corresponding game asset is loaded by the AssetManager. The `BuilderCodec` uses reflection to construct the object and populate its fields from the parsed asset data.
- **Scope:** The object's lifetime is bound to the asset that defines it. It typically persists as long as the world or game mode it belongs to is active. It is held as a field within a higher-level configuration registry or a World object.
- **Destruction:** The object is marked for garbage collection when the owning world or asset context is unloaded from memory. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state is mutable and consists of a series of boolean flags, all of which default to true. The state is intended to be fully established upon deserialization from an asset file and should be treated as effectively immutable thereafter.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms. It is designed to be populated on a loading thread and subsequently read from the main game thread. Concurrent modification will lead to unpredictable behavior and is strictly discouraged.

## API Surface
The public API consists solely of accessor methods to query the configured state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isDisplaySpawn() | boolean | O(1) | Returns true if the spawn point should be visible on the map. |
| isDisplayHome() | boolean | O(1) | Returns true if the player's home or bed location should be visible. |
| isDisplayWarps() | boolean | O(1) | Returns true if warp points or fast travel locations should be visible. |
| isDisplayDeathMarker() | boolean | O(1) | Returns true if the player's last death location should be marked. |
| isDisplayPlayers() | boolean | O(1) | Returns true if other players should be visible on the map. |

## Integration Patterns

### Standard Usage
The object should be retrieved from a world context or configuration manager and used for conditional rendering logic.

```java
// Example: Retrieving config from a hypothetical World object
WorldMapConfig mapConfig = world.getMapConfiguration();

if (mapConfig.isDisplayPlayers()) {
    // Logic to render player icons on the map UI
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldMapConfig()`. This bypasses the asset loading system and creates an object with default values, which will not reflect the actual configuration defined in the game assets. This can lead to subtle bugs where map features behave differently than designed.
- **Runtime Modification:** Do not modify the state of a WorldMapConfig object after it has been loaded. Other systems may have already cached its values. Configuration is meant to be loaded once and then treated as read-only.

## Data Pipeline
The primary flow for this object is from a physical file into an in-memory representation used by the game engine.

> Flow:
> World Asset File (.json, .hocon) -> Asset Loading Service -> **WorldMapConfig.CODEC** -> **WorldMapConfig Instance** -> World Logic / Map UI System<ctrl63>


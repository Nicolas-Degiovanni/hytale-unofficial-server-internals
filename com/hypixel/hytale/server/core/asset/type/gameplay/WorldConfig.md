---
description: Architectural reference for WorldConfig
---

# WorldConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Model

## Definition
```java
// Signature
public class WorldConfig {
```

## Architecture & Concepts
The WorldConfig class is a passive data structure that represents the gameplay rules and environmental parameters for a specific world instance. It is not a service or manager; rather, it is a model object that encapsulates configuration loaded from an external asset file.

The primary architectural feature of this class is the static **CODEC** field. This `BuilderCodec` instance defines a declarative mapping between the fields of the Java object and a structured data format, such as JSON. The engine's asset loading system uses this codec to perform serialization and deserialization, transforming an on-disk configuration file into a fully populated WorldConfig object in memory.

Once loaded, this object serves as the single source of truth for world-specific rules. Various server-side game systems, such as the time-of-day manager or the block interaction controller, query this object to enforce the intended gameplay behavior for the active world.

## Lifecycle & Ownership
- **Creation:** An instance of WorldConfig is created exclusively by the engine's `Codec` system during the server's world loading sequence. The static `CODEC` is invoked by the AssetManager, which handles the `new WorldConfig()` instantiation and subsequent field population from the asset data.
- **Scope:** The lifetime of a WorldConfig object is strictly tied to the lifetime of the server-side world instance it describes. It is created when a world is initialized and persists in memory as long as that world remains active.
- **Destruction:** The object is marked for garbage collection when the world it belongs to is unloaded by the server. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The object's state is highly mutable during the initial deserialization process managed by the `CODEC`. After this initial population, it is intended to be treated as an **effectively immutable** object. It holds no dynamic runtime state; it is a pure representation of the configuration defined at the time of loading.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms such as locks or volatile fields. However, the standard operational pattern—instantiate once during world load, then read from multiple game systems concurrently—makes it safe for use in a multi-threaded server environment.

    **WARNING:** Any external attempt to modify the fields of a shared WorldConfig instance after its initial creation is a severe anti-pattern. This will introduce race conditions and lead to unpredictable and inconsistent game behavior. Treat all instances as read-only after they are retrieved from the engine.

## API Surface
The public API consists entirely of accessors for retrieving configuration values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isBlockBreakingAllowed() | boolean | O(1) | Returns true if players are permitted to break blocks. |
| isBlockGatheringAllowed() | boolean | O(1) | Returns true if breaking blocks should yield resources. |
| isBlockPlacementAllowed() | boolean | O(1) | Returns true if players are permitted to place blocks. |
| getDaytimeDurationSeconds() | int | O(1) | Gets the duration of a full day cycle in real-world seconds. |
| getNighttimeDurationSeconds() | int | O(1) | Gets the duration of a full night cycle in real-world seconds. |
| getTotalMoonPhases() | int | O(1) | Retrieves the total number of distinct moon phases in the cycle. |
| getBlockPlacementFragilityTimer() | float | O(1) | Gets the time in seconds a newly placed block can be instantly broken. |
| getSleepConfig() | SleepConfig | O(1) | Returns the nested configuration object for sleep mechanics. |

## Integration Patterns

### Standard Usage
A game system should never create a WorldConfig instance. Instead, it should retrieve the pre-existing, fully-loaded instance from the current world's context.

```java
// Example from within a server-side game system
// Assumes 'worldContext' provides access to the current world's state

WorldConfig config = worldContext.getWorld().getConfig();

if (!config.isBlockPlacementAllowed()) {
    // Deny a block placement action and notify the player.
    return;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldConfig()`. This bypasses the asset loading and codec system, resulting in an object with default, unconfigured values. This will cause game logic to deviate from the intended world settings.

- **Runtime Modification:** Do not attempt to modify the state of a WorldConfig object after it has been loaded. Configuration is not designed to be changed dynamically at runtime. Changes must be made to the source asset files, followed by a server or world reload.

    ```java
    // ANTI-PATTERN: This will not work and is dangerous
    WorldConfig config = worldContext.getWorld().getConfig();
    // The 'allowBlockBreaking' field is protected and this pattern is unsupported.
    // Even if reflection were used, it would cause inconsistent state.
    ```

## Data Pipeline
The WorldConfig class is a terminal point in the asset loading pipeline and a starting point for gameplay logic systems.

> Flow:
> World Asset File (e.g., world.json) -> Server AssetManager -> **WorldConfig.CODEC** -> In-Memory **WorldConfig** Instance -> Game Systems (TimeOfDayManager, BlockInteractionSystem, etc.)

---


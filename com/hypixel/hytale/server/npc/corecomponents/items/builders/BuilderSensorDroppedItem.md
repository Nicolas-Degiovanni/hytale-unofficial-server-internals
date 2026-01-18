---
description: Architectural reference for BuilderSensorDroppedItem
---

# BuilderSensorDroppedItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items.builders
**Type:** Configuration Object & Factory

## Definition
```java
// Signature
public class BuilderSensorDroppedItem extends BuilderSensorBase {
```

## Architecture & Concepts

The BuilderSensorDroppedItem is a specialized configuration factory responsible for constructing and defining the behavior of a SensorDroppedItem. Within the Hytale NPC AI framework, "Sensors" are the primary mechanism by which an NPC perceives its environment. This specific builder translates a JSON configuration block into a live sensor that can detect dropped items.

Its architectural role is twofold:

1.  **Factory:** During the NPC asset loading phase, it consumes a JSON data structure, validates the input, and produces an instance of SensorDroppedItem.
2.  **Live Configuration Provider:** This is a critical design pattern. The builder is **not** discarded after creation. The resulting SensorDroppedItem instance maintains a permanent reference back to its builder. At runtime, the sensor queries this builder to retrieve its operational parameters (e.g., range, line of sight).

This pattern allows for dynamic value resolution. The builder's internal state is composed of various `Holder` types (e.g., DoubleHolder, BooleanHolder). These holders can resolve their values against a runtime ExecutionContext, enabling an NPC's sensing capabilities to change based on game state without requiring the sensor to be rebuilt or reconfigured.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderSensorDroppedItem is created by the server's NPC asset pipeline when it encounters the corresponding type in an NPC's JSON definition file. It is never instantiated directly in game logic.
-   **Scope:** The builder's lifetime is inextricably linked to the SensorDroppedItem it creates. It persists in memory as long as the sensor it built is part of an active NPC's behavior set.
-   **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded and all references to its SensorDroppedItem are released.

## Internal State & Concurrency

-   **State:** The builder's state is highly mutable during the initial configuration phase, driven by the `readConfig` method. After this phase, its internal structure is considered stable. However, the *values returned* by its accessor methods can be dynamic, as they are resolved at runtime through the `Holder` mechanism. The builder effectively acts as a cache for the parsed JSON configuration.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be operated on by a single thread associated with its NPC's AI processing tick. The `readConfig` method performs non-atomic state mutations. Concurrent access during or after initialization will lead to unpredictable behavior and is strictly unsupported.

## API Surface

The public API is primarily consumed by the NPC asset loading system and the SensorDroppedItem instance it produces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs a new SensorDroppedItem instance linked to this builder. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the JSON definition, populating and validating internal state. Throws exceptions on invalid data. |
| getRange(BuilderSupport) | double | O(1) | Retrieves the configured detection range, resolved against the current execution context. |
| getViewSectorRadians(BuilderSupport) | float | O(1) | Retrieves the view sector angle in radians. |
| getHasLineOfSight(BuilderSupport) | boolean | O(1) | Retrieves the line-of-sight requirement. |
| getItems(BuilderSupport) | String[] | O(1) | Retrieves the array of item name patterns to match. May return null. |
| getAttitudes(BuilderSupport) | EnumSet<Attitude> | O(N) | Retrieves and maps the configured sentiments to a set of core Attitude enums. |

## Integration Patterns

### Standard Usage

A game developer or designer does not interact with this class via Java code. Instead, they define its behavior declaratively in an NPC's JSON configuration file. The system handles the lifecycle internally.

```json
// Example NPC Behavior Definition
{
  "type": "SensorDroppedItem",
  "Range": 20.0,
  "ViewSector": 90.0,
  "LineOfSight": true,
  "Items": [
    "hytale:apple",
    "hytale:bread_*"
  ],
  "Attitudes": [
    "VALUABLE"
  ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BuilderSensorDroppedItem()`. The asset system is the sole owner of its creation process. Manually created instances will be unconfigured and cause runtime exceptions.
-   **State Re-configuration:** Do not call `readConfig` on a builder that is already attached to an active sensor. This will mutate the live configuration of an NPC and can lead to severe, unpredictable AI behavior.
-   **Instance Sharing:** A single builder instance must not be used to `build` multiple, distinct sensors. Each sensor requires its own unique configuration state.

## Data Pipeline

The flow of configuration data is unidirectional, from the static asset definition to the live, in-game sensor.

> Flow:
> NPC JSON File -> Server Asset Loader -> `readConfig` -> **BuilderSensorDroppedItem** (Internal State Populated) -> `build()` -> SensorDroppedItem Instance -> (Game Tick) -> Sensor calls `getRange`, `getItems`, etc. on its builder to perform world queries.


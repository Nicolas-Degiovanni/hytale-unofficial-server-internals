---
description: Architectural reference for ObjectiveLocationAreaRadius
---

# ObjectiveLocationAreaRadius

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.markerarea
**Type:** Configuration Model

## Definition
```java
// Signature
public class ObjectiveLocationAreaRadius extends ObjectiveLocationMarkerArea {
```

## Architecture & Concepts
The ObjectiveLocationAreaRadius class is a data-driven configuration object that defines a spherical area of interest for a game objective. It is a concrete implementation of the abstract ObjectiveLocationMarkerArea, specializing in radius-based spatial checks rather than other shapes.

Its primary role is to serve the server-side Adventure and Questing systems by providing the logic to determine if players are within specific zones centered around a marker position. The class is designed to be defined declaratively in external data files (e.g., JSON) and deserialized at runtime by the engine's core Codec system. The static CODEC field represents the public contract for this serialization, including strict validation rules.

Architecturally, this class implements a two-zone system:
1.  **Entry Area:** A smaller, inner sphere. A player entering this zone typically progresses an objective.
2.  **Exit Area:** A larger, outer sphere. A player must leave this larger zone to be considered fully "outside" the objective area.

This two-radius pattern creates a hysteresis effect, preventing rapid state-toggling if a player moves back and forth along the boundary of a single-radius zone.

## Lifecycle & Ownership
-   **Creation:** Instances are not meant to be instantiated directly with the *new* keyword in game logic. The Hytale Codec system is the designated creator, responsible for constructing and validating instances when loading adventure mode configurations from disk. The static CODEC field orchestrates this process, parsing keys like EntryRadius, validating their values, and running post-creation hooks.
-   **Scope:** The lifetime of an ObjectiveLocationAreaRadius instance is bound to its parent configuration object, typically a specific stage of a quest or a world objective. It persists in memory as long as the parent objective is active or defined.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once its parent configuration is unloaded, for example, when a quest is completed or a server zone is shut down. There are no explicit destruction methods.

## Internal State & Concurrency
-   **State:** The class holds a small, mutable state consisting of the entryArea and exitArea radii, along with pre-calculated bounding boxes inherited from its parent. **WARNING:** While technically mutable, this state is designed to be set only once during initialization. It should be treated as effectively immutable post-creation. The class does not cache the results of spatial queries.
-   **Thread Safety:** This class is **conditionally thread-safe**. Once an instance is fully constructed by the Codec system, its internal state does not change. Its methods are pure functions with respect to its own state, making it safe to be read by multiple threads. However, callers are responsible for ensuring that mutable arguments passed into its methods, such as the *results* list, are handled in a thread-safe manner.

## API Surface
The public API provides methods for querying player locations against the defined spherical zones.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayersInEntryArea(spatial, results, pos) | void | O(log N + k) | Populates the *results* list with all players inside the inner entry radius. Complexity depends on the spatial data structure. |
| getPlayersInExitArea(spatial, results, pos) | void | O(log N + k) | Populates the *results* list with all players inside the outer exit radius. |
| hasPlayerInExitArea(spatial, type, pos, buffer) | boolean | O(log N) | Performs an optimized check to see if at least one player exists within the exit radius. Often faster than collecting all players. |
| isPlayerInEntryArea(playerPosition, markerPosition) | boolean | O(1) | A fast, stateless geometric check to determine if a single point is within the entry radius. Does not require a spatial database. |

## Integration Patterns

### Standard Usage
This object should be retrieved from a higher-level configuration, such as an objective definition. Its methods are then used by server-side systems to update objective state based on player positions.

```java
// In a server-side objective-tracking system:
// Assume 'activeObjective' is the parent configuration object.

ObjectiveLocationMarkerArea areaConfig = activeObjective.getMarkerArea();
Vector3d markerPosition = activeObjective.getMarkerPosition();

// This pattern ensures type safety and correct behavior.
if (areaConfig instanceof ObjectiveLocationAreaRadius) {
    ObjectiveLocationAreaRadius radiusArea = (ObjectiveLocationAreaRadius) areaConfig;

    for (Player player : allActivePlayers) {
        boolean isInside = radiusArea.isPlayerInEntryArea(player.getPosition(), markerPosition);
        
        // Update the player's objective state based on the result.
        player.getObjectiveState().setInsideMarkerArea(isInside);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new ObjectiveLocationAreaRadius()`. This bypasses the crucial validation and post-processing steps defined in the CODEC, such as ensuring ExitRadius is greater than EntryRadius and pre-calculating the bounding boxes. This will lead to runtime errors or inconsistent behavior.
-   **State Mutation:** Do not modify the public fields of this class after it has been initialized. Altering `entryArea` or `exitArea` at runtime will de-synchronize them from the internal bounding boxes used for some spatial queries, resulting in unpredictable behavior.

## Data Pipeline
The flow of data for this class begins with a configuration file and ends with a game state update.

> Flow:
> Adventure Config File (JSON) -> Engine Codec Deserializer -> **ObjectiveLocationAreaRadius Instance** -> Objective System Logic -> SpatialResource Query -> Player Objective State Update


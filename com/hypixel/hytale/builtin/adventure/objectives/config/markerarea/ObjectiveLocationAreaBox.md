---
description: Architectural reference for ObjectiveLocationAreaBox
---

# ObjectiveLocationAreaBox

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.markerarea
**Type:** Transient

## Definition
```java
// Signature
public class ObjectiveLocationAreaBox extends ObjectiveLocationMarkerArea {
```

## Architecture & Concepts
The ObjectiveLocationAreaBox is a server-side data-oriented class that defines spatial trigger volumes for game objectives. It represents a specific strategy within the broader `ObjectiveLocationMarkerArea` system, implementing the concept of an objective area using two distinct, axis-aligned bounding boxes: an *entry area* and an *exit area*.

This class is a fundamental building block of the Adventure Mode system. It encapsulates the geometric logic required to determine if a player has entered or exited a region of interest associated with an objective marker. Its primary role is to act as a configuration object, deserialized from content files, that provides spatial query capabilities to the higher-level objective management systems.

The presence of a static `CODEC` field is a critical architectural pattern. It signifies that this class is designed for automated data binding, allowing game designers to define objective areas in configuration files (e.g., JSON) which are then loaded into the engine as fully-formed Java objects.

- **Entry Area:** A typically smaller box defining the region a player must enter to satisfy a condition, such as completing a "reach the location" objective.
- **Exit Area:** A typically larger box that contains the entry area. If a player leaves this outer boundary, the objective state might be reset or failed. This prevents players from "cheesing" an objective by repeatedly stepping in and out of the entry zone.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale `Codec` system during the server's content loading phase. The static `CODEC` field is used to deserialize configuration data into a new ObjectiveLocationAreaBox object. The `afterDecode` hook ensures internal state is correctly initialized post-deserialization. Manual instantiation via the constructor is rare and generally reserved for testing or procedural generation.
- **Scope:** The lifetime of an instance is bound to the specific objective it configures. It persists as long as the parent objective is active in the game world.
- **Destruction:** As a plain Java object with no native resources, it is managed by the garbage collector. It is eligible for collection once the objective that holds a reference to it is completed, unloaded, or otherwise destroyed.

## Internal State & Concurrency
- **State:** The class holds mutable state in its `entryArea` and `exitArea` fields. However, these are intended to be set only upon creation. The design encourages treating instances as immutable post-initialization. Methods like `getRotatedArea` reinforce this by returning a *new* instance rather than modifying the existing one.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. It is designed to be used by a single thread, typically the main server thread responsible for ticking the objective's logic. Accessing an instance from multiple threads without external synchronization will lead to unpredictable behavior. The owning objective system is responsible for ensuring safe publication and access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayersInEntryArea(spatial, results, pos) | void | Varies | Populates the `results` list with all players inside the entry area. Complexity depends on the underlying spatial data structure. |
| getPlayersInExitArea(spatial, results, pos) | void | Varies | Populates the `results` list with all players inside the exit area. Complexity depends on the underlying spatial data structure. |
| hasPlayerInExitArea(spatial, type, pos, buffer) | boolean | O(log N) | Performs an optimized check to see if the closest entity to the marker is a player within the exit area. |
| isPlayerInEntryArea(playerPos, markerPos) | boolean | O(1) | Performs a simple, fast geometric check to see if a point is inside the entry area. |
| getRotatedArea(yaw, pitch) | ObjectiveLocationMarkerArea | O(1) | Returns a **new** instance of the area, rotated around the Y-axis. Critical for objectives placed with a specific orientation. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly by most systems. It is a configuration object consumed by the objective management engine. The engine deserializes it and uses its methods to check player positions during game ticks.

```java
// Pseudo-code for an objective system's update loop
void checkObjective(Objective objective, Player player) {
    ObjectiveLocationAreaBox area = objective.getArea(); // Assumes the area is this type
    Vector3d markerPos = objective.getMarkerPosition();
    Vector3d playerPos = player.getPosition();

    // Rotate the area based on how the marker was placed in the world
    ObjectiveLocationMarkerArea rotatedArea = area.getRotatedArea(markerPos.getYaw(), 0);

    if (rotatedArea.isPlayerInEntryArea(playerPos, markerPos)) {
        // Trigger objective completion logic
        objective.complete();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not get a reference to the internal `entryArea` or `exitArea` and modify them after the object has been created. This violates the assumption of immutability and can cause difficult-to-trace bugs in spatial queries. Use `getRotatedArea` to create a transformed copy.
- **Ignoring Rotation:** Failing to use `getRotatedArea` for objective markers that are rotated in the world will result in incorrect, world-aligned trigger zones, leading to confusing gameplay where the visual marker does not match the logical trigger area.
- **Cross-Thread Access:** Do not share an instance across multiple threads without explicit locking. The objective system that owns this object is responsible for ensuring all access happens on a single, designated thread.

## Data Pipeline
The primary data flow for this class is from a configuration file into the game engine, where it is then used for spatial queries.

> Flow:
> Adventure Mode Config File -> Hytale Codec System -> **ObjectiveLocationAreaBox Instance** -> Objective System -> `isPlayerInEntryArea()` Query -> Boolean Result -> Game Event Bus


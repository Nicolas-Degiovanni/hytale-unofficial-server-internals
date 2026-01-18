---
description: Architectural reference for ObjectiveLocationMarkerArea
---

# ObjectiveLocationMarkerArea

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.markerarea
**Type:** Polymorphic Data Model

## Definition
```java
// Signature
public abstract class ObjectiveLocationMarkerArea {
```

## Architecture & Concepts
The ObjectiveLocationMarkerArea is an abstract base class that defines a geometric volume in the game world, used primarily by the Adventure Mode objective system. It serves as the contract for spatial triggers, enabling quest logic to react when players enter or exit a predefined area.

This class is a key component of a polymorphic design pattern, allowing quest designers to specify different shapes for objective areas within configuration files. The static **CODEC** field is the central mechanism for this pattern, responsible for deserializing configuration data into concrete implementations like **ObjectiveLocationAreaBox** or **ObjectiveLocationAreaRadius**.

Architecturally, this class acts as a bridge between static quest configuration and the dynamic server world state. It does not hold world state itself; instead, its methods operate on a **SpatialResource** passed in at runtime. This makes it a stateless and reusable data object for performing spatial queries against the server's entity database. Its primary responsibility is to answer the question: "Which players are currently inside this predefined volume?"

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the **new** keyword. They are instantiated by the Hytale serialization system via the static **CODEC** field. During server startup or when an adventure zone is loaded, quest configuration files are parsed, and the codec instantiates the appropriate subclass based on a "Type" identifier in the data.
- **Scope:** The lifetime of an ObjectiveLocationMarkerArea instance is tied to its parent objective configuration. It is loaded into memory when the objective becomes active and persists as long as the objective is being tracked by the server.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once the parent objective is completed, unloaded, or the server shuts down, and no other systems hold a reference to it. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The class is stateful but effectively immutable after initialization. It holds two **Box** fields, **entryAreaBox** and **exitAreaBox**, which define its maximum extents. These are calculated and set by the protected **computeAreaBoxes** method, which is expected to be called within the constructor of a concrete subclass. Once computed, these boxes are not intended to be modified.
- **Thread Safety:** This class is thread-safe for read operations. Because it does not contain references to mutable world state, its methods can be safely called from multiple threads.

    **WARNING:** While the object itself is thread-safe, the **SpatialResource** and **CommandBuffer** objects passed into its methods are not. All methods that query the world state, such as **getPlayersInEntryArea**, must be executed on the main server thread to prevent race conditions and world state corruption.

## API Surface
The public API is designed for querying player positions relative to the area's defined boundaries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayersInEntryArea(...) | void | O(log N + K) | Populates a provided list with all players inside the entry area. Complexity depends on the underlying SpatialResource implementation. |
| getPlayersInExitArea(...) | void | O(log N + K) | Populates a provided list with all players inside the exit area. |
| hasPlayerInExitArea(...) | boolean | O(log N) | Performs an optimized check to determine if at least one player is inside the exit area. |
| isPlayerInEntryArea(...) | boolean | O(1) | Performs a fast, purely mathematical check to see if a given coordinate is within the entry area. Does not query the world state. |
| getBoxForEntryArea() | Box | O(1) | Returns the pre-computed axis-aligned bounding box for the entry area. |
| getRotatedArea(yaw, pitch) | ObjectiveLocationMarkerArea | O(1) | Returns a representation of this area rotated by the given angles. The base implementation returns itself. |

## Integration Patterns

### Standard Usage
This object is typically held by a higher-level objective or quest manager. During the server tick, the manager queries the area to check for player presence and trigger game events.

```java
// Conceptual example within an ObjectiveManager
ObjectiveLocationMarkerArea triggerZone = currentObjective.getTriggerArea();
List<Ref<EntityStore>> playersInZone = new ArrayList<>();

// On each server tick, query the spatial resource
triggerZone.getPlayersInEntryArea(server.getSpatialResource(), playersInZone, objectiveOrigin);

if (!playersInZone.isEmpty()) {
    // Trigger "Objective Started" event for players in the list
    eventBus.post(new ObjectiveStartedEvent(playersInZone));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ObjectiveLocationAreaBox()`. This bypasses the deserialization process and may result in a partially initialized object. These objects are designed to be created from configuration data by the **CODEC**.
- **Cross-Thread World Queries:** Do not call methods like **getPlayersInEntryArea** from an asynchronous task or any thread other than the main server thread. This will cause severe concurrency issues with the world's spatial index.
- **Ignoring Origin:** The query methods require a **Vector3d** origin parameter. Failing to provide the correct world-space position of the objective will result in the spatial query being performed in the wrong location.

## Data Pipeline
The flow of data from configuration to in-game logic is a one-way pipeline.

> Flow:
> Quest Configuration File (JSON/HOCON) -> **CODEC** Deserializer -> **ObjectiveLocationMarkerArea Instance** -> Objective Tracking System -> Spatial Query on Server Tick -> Game Event Bus


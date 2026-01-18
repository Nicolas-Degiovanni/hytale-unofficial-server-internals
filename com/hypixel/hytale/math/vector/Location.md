---
description: Architectural reference for Location
---

# Location

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient

## Definition
```java
// Signature
public class Location {
```

## Architecture & Concepts
The Location class is a fundamental data structure that represents a complete spatial state within the game engine. It aggregates three critical pieces of information: a high-precision 3D position (Vector3d), a 3D orientation (Vector3f), and an optional world identifier (String).

This class serves as a more context-aware alternative to a simple Transform object. While a Transform defines position and rotation in an abstract space, a Location anchors that state to a specific, named game world. This is essential for systems managing entities across multiple dimensions, instances, or server shards.

The choice of a double-precision Vector3d for position is a deliberate design decision to mitigate floating-point precision issues that arise in large, open-world environments. This ensures that entity positions remain accurate even at extreme distances from the world origin. The rotation is stored as a standard single-precision Vector3f, representing pitch, yaw, and roll, which is sufficient for orientation.

A null value for the world field typically signifies a location in a context where the world is implied or irrelevant, such as in an item inventory or during the definition of a template.

## Lifecycle & Ownership
-   **Creation:** Location objects are instantiated on-demand by any system that needs to represent a point in a world. Common creators include entity spawning logic, network packet deserializers, and player movement controllers. There is no centralized factory or manager for Location objects.
-   **Scope:** The lifetime of a Location instance is bound to its owner. If it is a field within an Entity object, it lives and dies with that entity. If created as a temporary variable for a calculation, it is short-lived and eligible for garbage collection once out of scope.
-   **Destruction:** Memory is managed by the Java Garbage Collector. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
-   **State:** This class is **Mutable**. Its core fields—world, position, and rotation—can be modified after construction via public setters. This design prioritizes performance and ease of use by allowing in-place modification, avoiding the overhead of creating new objects for every state change.

-   **Thread Safety:** The Location class is **not thread-safe**. It contains no internal synchronization mechanisms like locks or volatile fields. Concurrent modification of a Location instance from multiple threads will result in race conditions, memory visibility issues, and unpredictable behavior.

    **WARNING:** All access to a shared Location instance must be externally synchronized. It is strongly recommended to confine its use to a single thread, such as the main game loop, or to pass immutable copies between threads.

## API Surface
The public API provides methods for state management and for deriving spatial information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDirection() | Vector3d | O(1) | Calculates and returns a normalized direction vector based on the current pitch and yaw. |
| getAxisDirection() | Vector3i | O(1) | Calculates a cardinal direction vector (e.g., (1,0,0)) by rounding the direction vector. Throws IllegalStateException if pitch or yaw is NaN. |
| getAxis() | Axis | O(1) | Derives the primary cardinal axis (X, Y, or Z) from the axis direction. |
| toTransform() | Transform | O(1) | Creates a new Transform object from the location's position and rotation, discarding the world identifier. |

## Integration Patterns

### Standard Usage
Location is primarily used as a data container to track and update the state of game objects. It is typically held as a field within an entity or component.

```java
// Example: Positioning a newly spawned entity
Entity myEntity = world.spawnEntity(EntityType.CREEPER);
Vector3d spawnPosition = new Vector3d(150.5, 64.0, -233.2);
Vector3f defaultRotation = new Vector3f(0.0f, 90.0f, 0.0f);

// Create a complete Location object
Location spawnLocation = new Location("world_overworld", spawnPosition, defaultRotation);

// Assign the location to the entity
myEntity.setLocation(spawnLocation);
```

### Anti-Patterns (Do NOT do this)
-   **Shared Mutable References:** Do not pass the same Location instance to multiple independent systems. Because the object is mutable, one system can inadvertently alter the state used by another, leading to extremely difficult-to-debug bugs. If you need to provide location data, pass a copy.
    ```java
    // BAD: Both systems operate on the SAME object
    Location sharedLoc = player.getLocation();
    physicsSystem.predict(sharedLoc);
    aiSystem.planPath(sharedLoc); // State may have been changed by physics!

    // GOOD: Provide a copy to ensure isolation
    Location aiLoc = new Location(player.getWorld(), player.getPosition(), player.getRotation());
    aiSystem.planPath(aiLoc);
    ```
-   **Concurrent Modification:** Never modify a Location object from one thread while another thread is reading it. This is a classic race condition.
    ```java
    // BAD: Network thread writes while game thread reads
    // Network Thread
    player.getLocation().setPosition(newPosition);

    // Main Game Thread
    renderer.draw(player.getLocation()); // May render a partially updated or corrupt position
    ```
-   **Ignoring NaN Rotation:** The internal state allows for rotation values to be NaN, often during initial construction. Calling methods like `getDirection` or `getAxisDirection` with a NaN rotation will cause a crash. Always ensure rotation is properly initialized before performing directional calculations.

## Data Pipeline
A Location object is not a processing component itself but rather the data payload that flows between major engine systems. It acts as the canonical representation of an object's spatial state.

> **Flow: Entity State Update**
>
> Player Input -> Movement Logic -> Updates Entity's **Location** object -> Network Serializer -> Outgoing Packet
>
> **Flow: World State Replication**
>
> Incoming Packet -> Network Deserializer -> Creates new **Location** object -> World State Manager -> Updates local Entity's **Location** -> Physics & Rendering Systems<ctrl63>


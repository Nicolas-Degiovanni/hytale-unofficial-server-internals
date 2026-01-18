---
description: Architectural reference for RelativeWaypointDefinition
---

# RelativeWaypointDefinition

**Package:** com.hypixel.hytale.builtin.path.waypoint
**Type:** Transient

## Definition
```java
// Signature
public class RelativeWaypointDefinition {
```

## Architecture & Concepts
The RelativeWaypointDefinition is an immutable value object that represents a single segment of a movement path. It is a fundamental data structure used by the entity pathfinding and movement systems.

Unlike an absolute waypoint which defines a specific coordinate in the world, this class defines a waypoint *relative* to the entity's current position and orientation. It encapsulates a simple instruction: "turn by a given rotation, then move forward a specific distance".

This approach is highly effective for defining pre-scripted or procedural paths that are independent of their starting location in the world. A collection of these objects, typically in a list, forms a complete path that a `PathExecutor` or similar system can interpret and apply to an entity's transform.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by path-building logic. This typically occurs when a higher-level system, such as a `PathFactory`, parses a path from a configuration asset or generates one procedurally. Direct instantiation using `new` is the intended mechanism.
- **Scope:** The lifetime of a RelativeWaypointDefinition is strictly tied to its containing `Path` object. It is a short-lived, lightweight object.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as the `Path` that references it is no longer in scope. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields `rotation` and `distance` are declared as `final` and are set only once during construction. The object's state cannot be modified after creation.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability, an instance can be safely read by multiple threads concurrently without any need for external synchronization or locks. This makes it safe to share path definitions across different systems, such as AI and physics threads.

## API Surface
The public contract is minimal, consisting of the constructor and two accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RelativeWaypointDefinition(rotation, distance) | constructor | O(1) | Constructs a new waypoint definition. |
| getRotation() | float | O(1) | Returns the relative rotation in degrees. |
| getDistance() | double | O(1) | Returns the travel distance in world units. |

## Integration Patterns

### Standard Usage
A RelativeWaypointDefinition is almost never used in isolation. It is created as part of a larger collection that defines a complete movement sequence.

```java
// A Path is typically composed of a list of relative waypoints.
List<RelativeWaypointDefinition> pathSegments = new ArrayList<>();

// Instruction: Turn 45 degrees right, move 10 units forward.
pathSegments.add(new RelativeWaypointDefinition(45.0f, 10.0));

// Instruction: Turn 90 degrees left, move 5 units forward.
pathSegments.add(new RelativeWaypointDefinition(-90.0f, 5.0));

// This list would then be passed to a movement controller.
entity.getMovementController().followPath(pathSegments);
```

### Anti-Patterns (Do NOT do this)
- **Misinterpretation of Data:** Do not treat the `rotation` and `distance` values as absolute world coordinates. They are relative instructions that must be interpreted by a path-following system in the context of an entity's current state.
- **State Modification:** Do not attempt to modify the internal state of this object using reflection. The immutability guarantee is critical for system stability and thread safety. Breaking this contract will lead to unpredictable behavior in the movement system.

## Data Pipeline
This class acts as a data payload within the entity movement pipeline. It is the raw, uninterpreted instruction that is passed from path definition to execution.

> Flow:
> Path Asset (JSON/XML) -> Asset Parser -> **RelativeWaypointDefinition** -> Path Object (`List`) -> AI Behavior Tree -> Movement System -> Entity Transform Update


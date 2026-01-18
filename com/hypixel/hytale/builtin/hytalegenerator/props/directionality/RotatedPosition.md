---
description: Architectural reference for RotatedPosition
---

# RotatedPosition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Value Object / Utility

## Definition
```java
// Signature
public class RotatedPosition {
```

## Architecture & Concepts
RotatedPosition is an immutable value object that represents a coordinate in 3D space coupled with an orientation. It is a fundamental data structure within the procedural world generation system, specifically for composing and placing prefabs.

Unlike a simple Vector3i which only defines a location, RotatedPosition also defines a "facing". This is critical for algorithms that need to place structures relative to one another. For example, calculating the world position of a doorway (child) that is defined by a local offset from the center of a room (parent).

The core architectural purpose of this class is to encapsulate the mathematics of hierarchical transformations. The `getRelativeTo` method provides a clean, declarative way to transform a local-space position and rotation into a parent's coordinate system, yielding a new world-space position and rotation. This enables complex, multi-part structures to be assembled procedurally without manual matrix math.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by world generation algorithms whenever a relative placement calculation is required. It is not managed by a service locator or dependency injection framework.
- **Scope:** Ephemeral. An instance of RotatedPosition typically exists only for the duration of a single calculation. It is passed as a parameter or returned from a method and is immediately eligible for garbage collection once its data has been consumed.
- **Destruction:** Handled automatically by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this class.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields (`x`, `y`, `z`, `rotation`) are declared `public final` and are set only once during construction. Methods that perform transformations, such as `getRelativeTo`, do not modify the internal state; they return a new RotatedPosition instance with the calculated result.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its immutability, a RotatedPosition object can be safely passed between, and read by, multiple threads without any risk of data corruption or race conditions. No external locking or synchronization is required when using this class.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRelativeTo(RotatedPosition other) | RotatedPosition | O(1) | Transforms this local position into the coordinate system of the `other` position. Returns a new instance representing the final world-space position and rotation. |
| toVector3i() | Vector3i | O(1) | Returns a new Vector3i containing only the positional data, discarding the rotation component. Used for final world interactions. |

## Integration Patterns

### Standard Usage
The primary pattern is compositional transformation. To find the final world placement of a child object, you define its position relative to its parent and then transform it.

```java
// A room is placed at (100, 50, 200) facing North
PrefabRotation roomRotation = PrefabRotation.NORTH;
RotatedPosition roomPlacement = new RotatedPosition(100, 50, 200, roomRotation);

// A door is located at a local offset of (5, 0, 0) inside the room, with no additional rotation
RotatedPosition doorOffset = new RotatedPosition(5, 0, 0, PrefabRotation.IDENTITY);

// Calculate the final world position and rotation of the door
RotatedPosition finalDoorPlacement = doorOffset.getRelativeTo(roomPlacement);

// finalDoorPlacement now correctly represents the door's absolute world coordinates and orientation.
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify an instance of RotatedPosition after creation. Its immutability is a core design guarantee. If a different position is needed, create a new instance.
- **Incorrect Transformation Order:** The order of operations in `getRelativeTo` is critical. The expression `child.getRelativeTo(parent)` correctly transforms the child's local offset into the parent's world space. The reverse, `parent.getRelativeTo(child)`, will produce a mathematically nonsensical result for this use case.

## Data Pipeline
RotatedPosition acts as a computational node in the world generation data flow, not as a data processor in a pipeline. Its role is to transform positional data during the prefab composition stage.

> Flow:
> Parent Placement Data -> Child Local Offset Data -> **RotatedPosition.getRelativeTo()** -> Final Placement Data -> World Block Update API


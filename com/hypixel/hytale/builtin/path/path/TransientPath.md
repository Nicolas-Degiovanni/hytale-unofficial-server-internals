---
description: Architectural reference for TransientPath
---

# TransientPath

**Package:** com.hypixel.hytale.builtin.path.path
**Type:** Transient

## Definition
```java
// Signature
public class TransientPath implements IPath<SimplePathWaypoint> {
```

## Architecture & Concepts
The TransientPath class is a concrete, in-memory implementation of the IPath interface. Its primary role is to represent a sequence of waypoints that are generated dynamically at runtime and are not intended for persistence. The name *Transient* explicitly signals its ephemeral nature; it has no persistent identity, returning null for both its ID and name.

This class acts as a path data structure, constructed procedurally from a set of relative instructions. The static factory method, buildPath, is the core of its design. It consumes a queue of RelativeWaypointDefinition objects, interpreting them as sequential "turn" and "move forward" commands to build a complete, world-space path from a given origin.

This component is typically used by systems that require on-the-fly path generation, such as AI behavior trees, scripted cinematic sequences, or dynamic environmental effects. It transforms abstract, relative movement commands into a concrete list of world-space coordinates and rotations.

### Lifecycle & Ownership
- **Creation:** Instances are created and populated exclusively through the static factory method buildPath. The system that provides the initial transform and movement instructions is the effective owner of the resulting TransientPath object.
- **Scope:** The lifetime of a TransientPath instance is bound to the system that created it. It exists only as long as a reference is maintained and is typically short-lived, existing only for the duration of a single task (e.g., an NPC walking from point A to B).
- **Destruction:** The object is managed by the Java garbage collector. It is deallocated once it is no longer referenced. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The class maintains an internal ObjectArrayList of SimplePathWaypoint objects. This list is populated sequentially during the execution of the buildPath factory method.
- **Thread Safety:** This class is **not thread-safe**. The underlying waypoint list is not synchronized. All operations, especially the initial construction via buildPath, must be confined to a single thread. Concurrent access will lead to unpredictable behavior and potential exceptions. While getPathWaypoints returns an unmodifiable view, this only prevents external modification of the list, not internal modification from other methods on the same instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| buildPath(origin, rotation, instructions, scale) | IPath | O(N) | **Static Factory.** Constructs and returns a new TransientPath by processing a queue of N relative movement instructions. This is the primary entry point. |
| getPathWaypoints() | List | O(1) | Returns an unmodifiable view of the internal waypoint list. **Warning:** Attempting to modify the returned list will result in an UnsupportedOperationException. |
| length() | int | O(1) | Returns the total number of waypoints in the path. |
| get(index) | SimplePathWaypoint | O(1) | Retrieves the waypoint at the specified index. Throws IndexOutOfBoundsException if the index is invalid. |
| getId() | UUID | O(1) | Always returns null. Reinforces the non-persistent nature of the path. |
| getName() | String | O(1) | Always returns null. |

## Integration Patterns

### Standard Usage
The sole intended pattern for creating a TransientPath is through its static factory method. The caller prepares a queue of relative instructions and provides a starting transform.

```java
// 1. Define the starting point and rotation
Vector3d startPosition = new Vector3d(100, 64, 200);
Vector3f startRotation = new Vector3f(0, 90, 0);

// 2. Create a queue of relative movement commands
Queue<RelativeWaypointDefinition> instructions = new ArrayDeque<>();
instructions.add(new RelativeWaypointDefinition(10.0, 0.0f));   // Move 10 blocks forward
instructions.add(new RelativeWaypointDefinition(5.0, 45.0f));  // Turn 45 degrees, move 5 blocks
instructions.add(new RelativeWaypointDefinition(10.0, -90.0f)); // Turn -90 degrees, move 10 blocks

// 3. Build the path
IPath<SimplePathWaypoint> dynamicPath = TransientPath.buildPath(startPosition, startRotation, instructions, 1.0);

// 4. Consume the path (e.g., by an AI system)
aiController.followPath(dynamicPath);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TransientPath()`. While the constructor is public, it creates an empty and useless path object. The buildPath factory is the only method that correctly populates the instance.
- **Concurrent Construction:** Do not share the `instructions` queue between threads that are both calling buildPath. The queue is consumed (mutated) during path construction, leading to a severe race condition.
- **Post-Construction Modification:** The class is not designed to be modified after it has been built. Calling the protected-scope `addWaypoint` method after construction may lead to inconsistent path data. Treat instances as effectively immutable once returned from the factory.

## Data Pipeline
TransientPath acts as a data transformer, converting abstract instructions into a concrete data structure for consumption by other engine systems.

> Flow:
> RelativeWaypointDefinition Queue -> **TransientPath.buildPath()** -> TransientPath Instance -> Path Following System (AI, Cinematics)


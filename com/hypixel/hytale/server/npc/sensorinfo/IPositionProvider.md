---
description: Architectural reference for the IPositionProvider interface, a core contract for NPC sensory systems.
---

# IPositionProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IPositionProvider {
    // Methods defined here
}
```

## Architecture & Concepts
The IPositionProvider interface is a fundamental contract within the server-side AI sensory framework. It serves as an abstraction layer that decouples AI logic, such as pathfinding or targeting, from the concrete source of positional data. This allows a single AI behavior to operate on different types of targets without modification.

An implementation of IPositionProvider can represent:
*   The dynamic location of a game entity (e.g., a player or another NPC).
*   A static, fixed coordinate in the world (e.g., a guard post or a point of interest).
*   A calculated position (e.g., the last known location of a lost target).

By standardizing how positional data is accessed, the engine promotes reusable and modular AI components. The design heavily favors performance, notably through the `providePosition` method, which avoids memory allocation by populating a pre-existing Vector3d object. This is critical for systems that run every server tick for hundreds of NPCs.

### Lifecycle & Ownership
As an interface, IPositionProvider does not have its own lifecycle. Instead, the lifecycle of its *implementing objects* is dictated by the AI systems that create and manage them.

- **Creation:** Instances are typically created by higher-level AI components, such as a `Sensor` or a `BehaviorTreeNode`, in response to a specific stimulus. For example, a sensor detecting a hostile player would instantiate an `EntityPositionProvider` that tracks that player.
- **Scope:** The lifetime of a provider is tied to the duration of the AI task. It may be short-lived, existing only for the time it takes to investigate a sound, or it may persist as long as an NPC is actively engaged with a target.
- **Destruction:** The object is eligible for garbage collection when the managing AI system releases its reference. The `clear` method provides a mechanism for explicit state reset, allowing implementations to be pooled and reused to reduce object churn.

## Internal State & Concurrency
- **State:** Implementations are inherently **mutable**. Their internal state, representing a world position or a target entity, is expected to change frequently, often on every server tick.
- **Thread Safety:** Implementations are **not thread-safe** and are not designed to be. All interactions with an IPositionProvider must occur on the main server thread for the world in which the associated NPC exists. Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is designed for high-frequency access within the game loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasPosition() | boolean | O(1) | Checks if the provider currently holds a valid, resolvable position. |
| providePosition(Vector3d) | boolean | O(1) | Populates the supplied Vector3d with the current position. Returns false if no position is available. |
| getX() / getY() / getZ() | double | O(1) | Direct, single-axis accessors. May be less performant than a single `providePosition` call. |
| getTarget() | Ref<EntityStore> | O(1) | Returns a reference to the tracked entity, or null if the provider tracks a static point. |
| clear() | void | O(1) | Invalidates the provider's current state, causing subsequent calls to `hasPosition` to return false. |

## Integration Patterns

### Standard Usage
An AI behavior, such as a pathfinding task, retrieves a provider and uses it on each tick to update its goal. The use of a pre-allocated `Vector3d` is the required pattern to prevent performance degradation from memory allocation.

```java
// An AI system holds a provider for its current target
IPositionProvider targetPosition = npc.getBrain().getActiveTarget();
Vector3d destination = new Vector3d(); // Re-used every tick

// In the per-tick update loop
if (targetPosition != null && targetPosition.hasPosition()) {
    if (targetPosition.providePosition(destination)) {
        // The 'destination' vector is now populated
        pathfinder.setGoal(destination);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Cross-Tick Caching:** Do not call `getX()` once and assume the value is valid for subsequent ticks. The underlying position is volatile and must be re-queried on every tick.
- **Asynchronous Access:** Never access a provider from a separate thread. All method calls must be synchronized with the main server game loop.
- **Ignoring `hasPosition`:** Do not call `providePosition` without first checking `hasPosition`. Doing so may result in operating on stale or uninitialized data.

## Data Pipeline
The IPositionProvider acts as a conduit, transforming a high-level concept (a target) into raw data (a coordinate) for consumption by low-level systems.

> Flow:
> AI Sensor Detects Entity -> Creates **EntityPositionProvider** -> Behavior Tree Accesses Provider -> **IPositionProvider** Resolves EntityStore to Vector3d -> Pathfinding System Consumes Vector3d -> NPC Movement Component Executes Path


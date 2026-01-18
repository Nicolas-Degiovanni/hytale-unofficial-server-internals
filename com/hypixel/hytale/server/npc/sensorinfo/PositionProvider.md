---
description: Architectural reference for PositionProvider
---

# PositionProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Transient State Object

## Definition
```java
// Signature
public class PositionProvider extends InfoProviderBase implements IPositionProvider {
```

## Architecture & Concepts
The PositionProvider is a fundamental component within the server-side NPC Artificial Intelligence framework. Its primary function is to serve as a standardized container for a three-dimensional world coordinate. It abstracts the *source* of a position, allowing AI behaviors to operate on a location without needing to know if it originates from a dynamic entity, a static world point, or a procedurally calculated target.

Architecturally, it acts as a write-only, resettable data cache. An AI sensor can populate the provider by pointing it to a game entity, from which it extracts the raw coordinate data from the entity's TransformComponent. Crucially, the PositionProvider deliberately **does not** retain a reference to the source entity. This decoupling is a key design choice to prevent strong references, simplify the object's lifecycle, and avoid unintended side effects if the source entity is destroyed or becomes invalid.

The internal state is managed using primitive double types and a boolean validity flag. This avoids the overhead of object wrappers and garbage collection pressure, which is critical in a high-performance, tick-based simulation environment.

### Lifecycle & Ownership
-   **Creation:** A PositionProvider is not a global or long-lived service. It is instantiated on-demand by higher-level AI systems, typically during the configuration of a specific NPC sensor or behavior. It is common to find these objects created as part of a larger composite AI goal.
-   **Scope:** The object's lifetime is tightly coupled to its owning AI behavior. It persists as long as the behavior is active, and its state is frequently reset and updated on each server tick via the `clear` and `setTarget` methods.
-   **Destruction:** The object becomes eligible for garbage collection when the parent AI behavior is deactivated or the associated NPC entity is removed from the world. There is no explicit destruction method; its cleanup is managed by the Java garbage collector.

## Internal State & Concurrency
-   **State:** The PositionProvider is highly mutable. Its internal state consists of three double-precision floating-point numbers (x, y, z) and a boolean flag, `isValid`. The state is considered invalid by default, initialized with large sentinel values until explicitly set via a `setTarget` call.
-   **Thread Safety:** **This class is not thread-safe.** It is designed exclusively for use within the single-threaded context of the server's main entity update loop. All methods perform unsynchronized reads and writes to its internal fields. Concurrent access from multiple threads will result in race conditions, state corruption, and unpredictable AI behavior.

## API Surface
The public API is designed for high-frequency state updates and queries within the game loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTarget(Ref, ComponentAccessor) | Ref<EntityStore> | O(1) | Updates the internal position from a valid, living entity's TransformComponent. Clears the state if the entity is null, invalid, or dead. |
| setTarget(Vector3d) | void | O(1) | Sets the internal position from a given Vector3d object. Marks the position as valid. |
| setTarget(double, double, double) | void | O(1) | Sets the internal position from raw coordinates. Marks the position as valid. |
| providePosition(Vector3d) | boolean | O(1) | Fills the provided `result` vector with the stored coordinates. Returns false if no valid position is stored. |
| hasPosition() | boolean | O(1) | Returns true if the provider holds a valid position set by a `setTarget` call. |
| clear() | void | O(1) | Resets the provider to its initial, invalid state. |
| getTarget() | Ref<EntityStore> | O(1) | **WARNING:** Always returns null. This is by design to enforce decoupling. |

## Integration Patterns

### Standard Usage
The PositionProvider is intended to be used as a transient data holder within an AI update cycle. The typical pattern is to clear, set, and then read the position within the same logical scope.

```java
// Within an AI Behavior's update method...

// 1. Obtain the provider instance (owned by this behavior)
PositionProvider targetLocation = this.getTargetLocationProvider();

// 2. Clear previous state from the last tick
targetLocation.clear();

// 3. Attempt to acquire a new target entity
Ref<EntityStore> enemy = findNearestEnemy();
targetLocation.setTarget(enemy, world.getComponentAccessor());

// 4. Check validity before using the data
if (targetLocation.hasPosition()) {
    Vector3d pos = new Vector3d();
    targetLocation.providePosition(pos);
    
    // Use the position for navigation, attacks, etc.
    navigateTo(pos);
}
```

### Anti-Patterns (Do NOT do this)
-   **Assuming State:** Never call `providePosition` without first checking `hasPosition`. The coordinates will be meaningless sentinel values if the state is invalid.
-   **Cross-Thread Access:** Do not share a PositionProvider instance between threads or access it from asynchronous tasks. It must only be manipulated by the main server thread.
-   **Extending to Store References:** Do not subclass or modify this provider to store the `Ref<EntityStore>`. This violates the core architectural principle of decoupling the position data from its source. If you need both, use a separate variable.

## Data Pipeline
The PositionProvider acts as a simple state machine in the NPC sensory data pipeline. It transforms a reference to a complex game object into a simple, reusable coordinate.

> Flow:
> AI Sensor Logic -> `Ref<EntityStore>` -> `PositionProvider.setTarget()` -> **Internal State (x, y, z, isValid)** -> AI Behavior Logic -> `PositionProvider.providePosition()` -> `Vector3d` -> Pathfinding / Action System


---
description: Architectural reference for SensorAdjustPosition
---

# SensorAdjustPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorAdjustPosition extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts

The SensorAdjustPosition is a specialized component within the NPC sensing system that operates based on the **Decorator** design pattern. Its primary function is not to sense the world itself, but to intercept and modify the output of another, wrapped Sensor.

Architecturally, it serves as a composable filter in an NPC's behavior definition. It allows designers to reuse a generic sensor (e.g., one that finds the nearest player) and apply a spatial offset to its result without altering the original sensor's logic. For example, an NPC might need to target a position 2 meters *above* a player's head; this component achieves that by wrapping a player-finding sensor and applying a (0, 2, 0) offset.

It achieves this by:
1.  **Wrapping:** Holding a reference to an inner Sensor instance.
2.  **Delegating:** Forwarding all lifecycle method calls (e.g., `loaded`, `spawned`, `unloaded`) directly to the wrapped Sensor, making it a transparent proxy.
3.  **Modifying:** In its core `matches` method, it first validates that the wrapped Sensor's conditions are met. If they are, it retrieves the position data from the wrapped Sensor, applies its own configured `Vector3d` offset, and stores the result in its own internal `PositionProvider`.

This design promotes modularity and reusability within the NPC asset definition pipeline, preventing the need for numerous, slightly different Sensor implementations.

### Lifecycle & Ownership

-   **Creation:** SensorAdjustPosition is instantiated exclusively by the server's NPC asset loading system, typically via its corresponding builder, `BuilderSensorAdjustPosition`. It is constructed during the parsing of an NPC's behavior configuration, which is often defined in a declarative format like JSON. **WARNING:** Manual instantiation via `new` will result in a misconfigured and non-functional component.
-   **Scope:** The component's lifetime is strictly tied to the `Role` of its parent `NPCEntity`. It remains active as long as the NPC's current role is active and is discarded if the role changes.
-   **Destruction:** The component is marked for garbage collection when its parent `Role` is unloaded or destroyed. The `done()` method is invoked as part of this cleanup, which in turn delegates the call to the wrapped sensor.

## Internal State & Concurrency

-   **State:** This component is **mutable**. Its primary state is the calculated target position stored within its internal `positionProvider`. This state is volatile and is recalculated on every successful invocation of the `matches` method. The `offset` vector and the reference to the wrapped `sensor` are immutable after construction.
-   **Thread Safety:** **This component is not thread-safe.** It is designed to be owned and operated by a single thread: the server's main game loop that processes NPC updates. All method calls, especially `matches`, must be confined to this thread. Concurrent access from other threads will lead to race conditions and undefined behavior in the NPC's decision-making process.

## API Surface

The public API is primarily concerned with the NPC system's internal contracts for sensing and lifecycle management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(C) | Evaluates the wrapped sensor. If it matches, applies the vector offset to the result. C is the complexity of the wrapped sensor's `matches` method. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal provider containing the *adjusted* position. The data is only valid for the tick in which `matches` returned true. |
| loaded(role) | void | O(1) | Lifecycle hook. Delegates the call directly to the wrapped sensor. |
| spawned(role) | void | O(1) | Lifecycle hook. Delegates the call directly to the wrapped sensor. |
| unloaded(role) | void | O(1) | Lifecycle hook. Delegates the call directly to the wrapped sensor. |

## Integration Patterns

### Standard Usage

This component is not intended to be used imperatively in code. It is defined declaratively within an NPC's asset files. Game logic code would then interact with the *result* of the sensor evaluation, which is typically handled by a higher-level behavior tree or state machine.

A conceptual code example demonstrating how a system might consume the sensor's output:

```java
// In a hypothetical Behavior Tree node:
// This SensorAdjustPosition instance is a member of the node, configured at load time.
SensorAdjustPosition sensor = this.configuredSensor;

// During the NPC's update tick:
if (sensor.matches(entityRef, role, dt, store)) {
    // Retrieve the adjusted position
    InfoProvider info = sensor.getSensorInfo();
    IPositionProvider pos = info.getPositionProvider();

    // Use the adjusted position to command the NPC
    npc.getMotionController().moveTo(pos.getX(), pos.getY(), pos.getZ());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new SensorAdjustPosition()`. The component relies on the asset building pipeline for correct initialization of its offset and wrapped sensor. Failure to do so will result in a `NullPointerException`.
-   **State Caching:** Do not retrieve the `InfoProvider` and cache it for use across multiple game ticks. The position data is only guaranteed to be valid during the same tick that `matches` returned true. On subsequent ticks, the provider may be cleared or contain stale data.
-   **Manual Lifecycle Management:** Do not call methods like `loaded`, `spawned`, or `unloaded` directly. These are managed automatically by the parent `Role` and are critical for the correct state propagation to the wrapped sensor.

## Data Pipeline

The primary function of this component is to act as a transformation step in the NPC sensing data pipeline. It takes positional data from one sensor and outputs modified positional data for consumption by other systems.

> Flow:
> Wrapped Sensor Evaluation -> `matches()` returns true -> **SensorAdjustPosition** retrieves position -> Applies `Vector3d` offset -> Updates internal `PositionProvider` -> Downstream Consumer (e.g., Behavior Node) reads adjusted position


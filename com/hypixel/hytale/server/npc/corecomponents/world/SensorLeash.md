---
description: Architectural reference for SensorLeash
---

# SensorLeash

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorLeash extends SensorBase {
```

## Architecture & Concepts
The SensorLeash is a fundamental component within the server-side NPC artificial intelligence framework. Its primary responsibility is to act as a spatial trigger, detecting when an NPC has moved beyond a predefined distance from its designated anchor point, known as the leash point.

Architecturally, it functions as a **stateful predicate**. During an NPC's AI update cycle, the SensorLeash is evaluated. If the NPC is outside the allowed range, the sensor's `matches` method returns true. Critically, this evaluation is not a pure function; it has the side effect of populating an internal data provider with the coordinates of the leash point.

This design decouples the *detection* of a condition (being out of range) from the *reaction* to that condition (moving back). Other systems, such as a `MoveTo` behavior, can then query the sensor's `InfoProvider` to get the target location without needing to know the specifics of how that location was determined. This makes the AI system more modular and extensible.

## Lifecycle & Ownership
-   **Creation:** A SensorLeash instance is never created directly. It is instantiated by the NPC asset loading pipeline via its corresponding builder, `BuilderSensorLeash`. This occurs when an NPC's behavioral definition is parsed from configuration files and its component graph is constructed.
-   **Scope:** The lifecycle of a SensorLeash is tightly coupled to its parent `NPCEntity`. It exists for the entire duration that the NPC is active in the game world.
-   **Destruction:** The object is eligible for garbage collection when its parent `NPCEntity` is despawned and removed from the world. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** This component is **stateful**.
    -   The `range` and `rangeSq` fields are immutable after construction, defined by the NPC's asset configuration.
    -   The `positionProvider` field holds mutable state that is updated on every call to the `matches` method. It effectively caches the result of the last positive sensor check, storing the target leash point only when the NPC is out of range. When the NPC is within range, this provider is cleared.

-   **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed to be owned and operated by a single NPC's AI controller within the main server game loop. All method calls must be synchronized with the server tick. Unmanaged concurrent access will lead to race conditions on the internal `positionProvider` state, causing unpredictable AI behavior.

## API Surface
The public contract is minimal, focusing exclusively on evaluation and data retrieval within a single AI tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC is outside its leash range. Mutates internal state by populating the PositionProvider on a positive match. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal PositionProvider, which contains the target leash point if `matches` returned true during the same tick. |

## Integration Patterns

### Standard Usage
The SensorLeash is designed to be polled by an NPC's `Role` or an equivalent AI scheduler during each update tick. If the sensor triggers, the scheduler retrieves the `InfoProvider` and activates a corresponding behavior to act on the data.

```java
// Within an NPC's AI update logic
SensorLeash leash = npc.getSensor(SensorLeash.class);

// The matches method has a side-effect on the sensor's internal state
if (leash.matches(npcRef, currentRole, deltaTime, entityStore)) {
    // Retrieve the data populated by the successful matches() call
    InfoProvider info = leash.getSensorInfo();

    // Activate a behavior to return the NPC to the leash point
    behaviorScheduler.startBehavior(new ReturnToLeashPointBehavior(info));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new SensorLeash()`. The component's `range` is configured through a builder during asset loading. Direct instantiation will result in a misconfigured and non-functional sensor.
-   **State Caching:** Do not cache the `InfoProvider` returned by `getSensorInfo` across multiple ticks. The provider's state is only guaranteed to be valid for the single tick in which the `matches` method returned true. It is cleared on subsequent evaluations.
-   **External State Modification:** Do not attempt to modify the `PositionProvider` obtained from `getSensorInfo`. It is intended as a read-only data source for other AI systems.

## Data Pipeline
The flow of data through this component is unidirectional and tied to the server's AI tick.

> Flow:
> Server AI Tick -> `Role` polls sensors -> **SensorLeash.matches()** calculates distance -> (If out of range) -> Internal `PositionProvider` is populated with leash point -> `Role` calls `getSensorInfo()` -> `InfoProvider` data is passed to a `MoveTo` Behavior -> Pathfinding System


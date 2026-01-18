---
description: Architectural reference for SensorNot
---

# SensorNot

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorNot extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts

The SensorNot class is a fundamental logical component within the server-side NPC Behavior System. It functions as a **Decorator** that wraps another Sensor instance, effectively inverting its activation condition. This provides a logical NOT operator, enabling the creation of more complex and nuanced NPC behaviors.

Architecturally, SensorNot is a composite node in an NPC's behavior tree or state machine. Its primary role is to answer the question: "Is a specific condition **not** met?". For example, it can be used to check if a target is *not* visible, if an NPC is *not* in water, or if a specific item is *not* in a character's inventory.

It integrates directly with the NPC's memory and targeting system, known as `MarkedEntitySupport`. When its condition is met (meaning the wrapped sensor's condition fails), it can provide positional information about a relevant entity to other systems. Conversely, when its condition fails, it can automatically clear a previously marked target, preventing stale data from influencing other behaviors.

This component is crucial for creating defensive, opportunistic, or state-aware AI that must react to the absence of a stimulus as much as its presence.

### Lifecycle & Ownership

The lifecycle of a SensorNot instance is managed entirely by the NPC asset pipeline and the parent `Role` component. It is not intended for manual instantiation or management.

-   **Creation:** A SensorNot is instantiated by the NPC asset loader via its corresponding builder, `BuilderSensorNot`. This occurs when the server parses an NPC's behavior definition file (e.g., a JSON asset) and constructs the in-memory behavior graph for a specific NPC type.
-   **Scope:** The component's lifetime is bound to the `Role` instance of the NPC it belongs to. It is created once when the `Role` is initialized and persists as long as the NPC entity exists and has that behavior set.
-   **Destruction:** The instance is eligible for garbage collection when its parent `Role` is destroyed. This typically happens when the NPC entity is unloaded or removed from the world. The `done` method is called as part of this teardown, which propagates the event to the wrapped sensor, allowing for proper resource cleanup.

## Internal State & Concurrency

-   **State:** Mutable. SensorNot maintains an internal `EntityPositionProvider` which caches positional data when the sensor is active. The state of this provider is updated on each call to the `matches` method. The core logical state, however, is transient and re-evaluated each tick based on the result from the wrapped `Sensor`.

-   **Thread Safety:** **Not thread-safe.** All operations on a SensorNot instance are expected to occur on the single thread responsible for updating the NPC's world. The `matches` method reads from and interacts with the `EntityStore`, which is an inherently single-threaded data structure within the context of a game tick. Unsynchronized access from other threads will lead to data corruption and undefined behavior.

## API Surface

The public API is primarily composed of the `matches` method and lifecycle hooks delegated from the `Sensor` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(S) | Returns true if the wrapped sensor returns false. S is the complexity of the wrapped sensor. This is the core evaluation method. |
| getSensorInfo() | InfoProvider | O(1) | Provides access to cached positional data when the sensor is active. |
| registerWithSupport(role) | void | O(S) | Lifecycle hook. Propagates the registration call to the wrapped sensor. |
| loaded(role) | void | O(S) | Lifecycle hook. Propagates the loaded event to the wrapped sensor. |
| spawned(role) | void | O(S) | Lifecycle hook. Propagates the spawned event to the wrapped sensor. |

**Warning:** All other public methods are lifecycle events (`unloaded`, `removed`, `teleported`, `done`) or component introspection methods (`componentCount`, `getComponent`). These are called exclusively by the parent `Role` and should not be invoked by user code.

## Integration Patterns

### Standard Usage

A developer does not interact with SensorNot directly in Java code. Instead, it is defined declaratively within an NPC's behavior asset file. The game's `Role` system is responsible for invoking the `matches` method during the behavior tree evaluation each tick.

The following pseudo-code illustrates how SensorNot would be used to make a guard NPC patrol only when a player is **not** visible.

```yaml
# Example NPC Behavior Definition (Conceptual)
behaviors:
  - priority: 1
    trigger:
      # This entire block defines the condition for the behavior.
      # The 'selector' will activate if ANY of its children are true.
      selector:
        - sequence: # Attack sequence
            - sensor: IsPlayerVisible
            - instruction: AttackTarget
        - sequence: # Patrol sequence
            - sensor:
                # The SensorNot inverts the IsPlayerVisible check.
                # This sequence only runs if the player is NOT visible.
                type: SensorNot
                sensor: IsPlayerVisible
            - instruction: FollowPatrolPath
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new SensorNot()`. The component requires a builder and a `BuilderSupport` context to be properly configured with its wrapped sensor and target slot information. Manual creation will result in a non-functional component that throws NullPointerExceptions.

-   **State Manipulation:** Do not manually call methods on the `EntityPositionProvider` returned by `getSensorInfo`. The internal state of SensorNot is managed exclusively by its `matches` method and should be treated as read-only by external systems.

-   **Lifecycle Mismanagement:** Overriding lifecycle methods like `loaded` or `spawned` in a subclass without calling the `super` implementation is a critical error. This breaks the decorator pattern and prevents the wrapped sensor from receiving essential lifecycle events, leading to memory leaks or incorrect behavior.

## Data Pipeline

The primary data flow through SensorNot is for logical evaluation within the NPC behavior system. It transforms a positive condition check into a negative one.

> Flow:
> NPC Behavior Tree Processor -> **SensorNot.matches()** -> Calls `matches()` on wrapped Sensor -> Inverts boolean result -> Returns `true` or `false` to Behavior Tree Processor -> Behavior Tree state transition (e.g., activates a different branch)


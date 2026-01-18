---
description: Architectural reference for TimeoutCondition
---

# TimeoutCondition

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** Transient

## Definition
```java
// Signature
public class TimeoutCondition implements RemovalCondition {
```

## Architecture & Concepts
The TimeoutCondition is a concrete implementation of the **Strategy Pattern**, conforming to the RemovalCondition interface. Its sole responsibility is to determine if a game world instance should be removed based on a pre-configured time duration.

This class encapsulates a single, self-contained rule: "shut down this world after N seconds have elapsed since the first time this condition was checked". It is designed to be configured and then evaluated periodically by a higher-level world instance management system.

Crucially, the TimeoutCondition object itself is stateless after its initial configuration. The state it relies upon—the calculated expiration timestamp—is not stored within the object itself. Instead, it is written to and read from an `InstanceDataResource` associated with the specific world being evaluated. This design allows a single configured TimeoutCondition to be conceptually applied to multiple worlds, while each world maintains its own independent expiration timer.

## Lifecycle & Ownership
- **Creation:** A TimeoutCondition instance is typically created in one of two ways:
    1. **Deserialization:** Instantiated by the Hytale codec system from configuration data (e.g., a world template JSON file) using its static `CODEC` field. This is the primary mechanism for defining world behavior.
    2. **Programmatic Instantiation:** Created directly via `new TimeoutCondition(double seconds)` by server logic that needs to dynamically construct removal rules.

- **Scope:** The object's lifetime is typically tied to the in-memory representation of a world's configuration. It is read during the setup of a world's removal policy and may be discarded afterward, as the critical state (the expiration time) is persisted in the world's `InstanceDataResource`.

- **Destruction:** The object is eligible for garbage collection once the system that configured the removal policy no longer holds a reference to it.

## Internal State & Concurrency
- **State:** The TimeoutCondition object is **effectively immutable** after construction. Its `timeoutSeconds` field is set once and never modified. However, it is a manipulator of external state. The first time `shouldRemoveWorld` is called for a given world, it performs a write operation to the world's `InstanceDataResource` to set the expiration timer. Subsequent calls are read-only.

- **Thread Safety:** This class is **not thread-safe**. The `shouldRemoveWorld` method contains a check-then-act sequence on the external `InstanceDataResource` (`getTimeoutTimer` followed by `setTimeoutTimer`). If multiple threads were to call this method for the same world instance simultaneously before the timer is initialized, a race condition would occur, potentially setting multiple different expiration times.

    **WARNING:** The engine's world management system MUST guarantee that all `RemovalCondition` evaluations for a specific world instance are serialized and executed on a single, designated thread, such as the world's main tick thread. Concurrent invocation will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TimeoutCondition() | constructor | O(1) | Creates a condition with a default 5-minute timeout. |
| TimeoutCondition(double) | constructor | O(1) | Creates a condition with a specified timeout in seconds. |
| shouldRemoveWorld(Store) | boolean | O(1) | Evaluates the condition. Returns true if the timeout has expired. |

## Integration Patterns

### Standard Usage
This component is not intended to be used in isolation. It is designed to be provided to a world instance manager which periodically evaluates it as part of the world's update cycle.

```java
// In a world management or configuration context

// 1. Create the condition, often from a config file
RemovalCondition condition = new TimeoutCondition(300); // 5 minutes

// 2. A world manager periodically calls the method on the world's tick
// boolean needsRemoval = condition.shouldRemoveWorld(world.getStore());
// if (needsRemoval) {
//    world.scheduleForShutdown();
// }
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Evaluation:** Do not invoke `shouldRemoveWorld` from multiple threads for the same world instance. The internal lazy-initialization of the timer is not atomic and will cause race conditions.
- **External State Mutation:** Do not manually retrieve the `InstanceDataResource` and modify the timeout timer from other systems. This will break the deterministic logic of the condition and lead to unpredictable world shutdown behavior.

## Data Pipeline
The TimeoutCondition acts as a gate in a control flow rather than a step in a data pipeline.

> Flow:
> World Tick -> World Instance Manager -> **TimeoutCondition.shouldRemoveWorld(store)** -> Boolean Result -> Manager schedules shutdown


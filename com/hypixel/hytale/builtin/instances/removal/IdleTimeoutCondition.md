---
description: Architectural reference for IdleTimeoutCondition
---

# IdleTimeoutCondition

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** Transient Policy Object

## Definition
```java
// Signature
public class IdleTimeoutCondition implements RemovalCondition {
```

## Architecture & Concepts
The IdleTimeoutCondition is a specific implementation of the RemovalCondition strategy interface. It encapsulates a single rule for determining if a game world instance should be shut down to conserve server resources: whether the world has been empty of players for a configured duration.

This component is designed to be data-driven. Its behavior, specifically the timeout duration, is configured and instantiated by the engine's serialization system via its static CODEC field. This allows server administrators to define and tune world removal policies without modifying engine source code.

Architecturally, this class is a stateless predicate. It does not maintain any internal state related to the timer or player presence. Instead, it operates on state stored externally within a world's resource stores, primarily the InstanceDataResource and TimeResource. This separation of logic (the condition) from state (the timer data) is a critical design pattern that ensures the condition itself remains simple, reusable, and thread-safe.

## Lifecycle & Ownership
- **Creation:** Instances are created by the Hytale Codec system during the deserialization of server or world configuration files. It is not intended for manual instantiation. A higher-level service, such as a WorldInstanceManager, will hold a collection of RemovalCondition objects, including this one.
- **Scope:** The object's lifetime is tied to the server configuration it was loaded from. It persists as long as the world instance profile it belongs to is active.
- **Destruction:** The object is eligible for garbage collection when its parent configuration is reloaded or the server shuts down. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its only internal field, timeoutSeconds, is set once upon creation by the codec and is never modified thereafter. All operational state, such as the calculated time when a world should be removed, is stored and managed externally in the InstanceDataResource.

- **Thread Safety:** The class is inherently thread-safe due to its stateless and immutable nature. The `shouldRemoveWorld` method is safe to call from any thread. However, the method performs read-modify-write operations on the external InstanceDataResource. Therefore, the overall thread safety of the operation is dependent on the concurrency guarantees provided by the underlying Store and Resource systems. It is assumed that these engine-level components provide the necessary synchronization.

## API Surface
The public contract is defined by the RemovalCondition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldRemoveWorld(Store) | boolean | O(1) | Evaluates if the world should be removed. If no players are present, it sets or checks an idle timer stored in an InstanceDataResource. Returns true if the timer has expired. |

## Integration Patterns

### Standard Usage
This condition is not invoked directly. It is evaluated by a higher-level world management system that iterates through a list of configured RemovalConditions for a given world instance.

```java
// Pseudo-code for a world management service
List<RemovalCondition> conditions = world.getConfig().getRemovalConditions();
Store<ChunkStore> worldStore = world.getChunkStore().getStore();

for (RemovalCondition condition : conditions) {
    if (condition.shouldRemoveWorld(worldStore)) {
        // Initiate world shutdown sequence
        world.scheduleForRemoval("Condition met: " + condition.getClass().getSimpleName());
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IdleTimeoutCondition()`. This bypasses the configuration-driven `CODEC` and will result in the object using a hardcoded default timeout, ignoring server-specific settings.
- **External State Manipulation:** Avoid reading from or writing to the `IdleTimeoutTimer` field within the `InstanceDataResource` from any other system. This class is the sole manager of that state, and external interference will lead to unpredictable world removal behavior.

## Data Pipeline
The IdleTimeoutCondition acts as a predicate in a control flow rather than a step in a data transformation pipeline. Its function is to produce a boolean decision based on engine state.

> Flow:
> World Management Service (Tick) -> **IdleTimeoutCondition.shouldRemoveWorld()** -> Reads World Player Count & TimeResource -> Reads/Writes InstanceDataResource (Timer) -> Returns boolean decision -> World Management Service acts on decision


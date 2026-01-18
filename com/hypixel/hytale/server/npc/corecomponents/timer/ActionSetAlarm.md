---
description: Architectural reference for ActionSetAlarm
---

# ActionSetAlarm

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetAlarm extends ActionBase {
```

## Architecture & Concepts
The ActionSetAlarm class is a server-side, data-driven command object used within the Non-Player Character (NPC) behavior system. It represents a single, atomic action that an NPC can perform: setting or clearing a timer on itself.

Its primary role is to bridge the declarative NPC behavior assets (e.g., behavior trees or state machines defined in configuration files) with the imperative game state. When an NPC's logic dictates that a timer should be started, an instance of this class is created and executed. It encapsulates the logic for calculating a future timestamp—based on a configured duration range and the current world time—and writing that value to an Alarm component associated with the NPC entity.

This class is a key component for implementing time-based behaviors, such as cooldowns, delayed reactions, or scheduled state transitions, without coupling the high-level behavior logic to the low-level details of time management or entity-component storage.

## Lifecycle & Ownership
- **Creation:** ActionSetAlarm is instantiated by the server's asset loading and NPC behavior systems, specifically via its corresponding builder, BuilderActionSetAlarm. It is **not** intended for direct manual instantiation in gameplay code. The builder populates the instance with configuration (e.g., duration, target alarm) defined in an NPC asset file.
- **Scope:** The object's lifetime is extremely short, typically confined to a single game tick. It is created, its execute method is called once, and it is then immediately eligible for garbage collection. It is a stateless, transient command.
- **Destruction:** The object is automatically garbage collected after the `execute` method completes. No manual cleanup is required. Holding references to an ActionSetAlarm instance across multiple ticks is an architectural violation.

## Internal State & Concurrency
- **State:** Immutable. All internal fields, including the target Alarm, minimum duration, and random variation, are marked as final and are set exclusively during construction. The object itself does not maintain any state related to the game world; it only carries its configuration.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. However, the `execute` method is **not** safe to call from arbitrary threads. It performs write operations on the EntityStore, which is a shared, mutable resource. All invocations of `execute` must be synchronized with the main server game loop to prevent data corruption and race conditions. The engine's NPC processing loop is expected to enforce this guarantee.

## API Surface
The public API is minimal, consisting of the `execute` method inherited from its parent, ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Sets or cancels the configured Alarm on the target entity. Retrieves current time from the WorldTimeResource and writes the calculated future time to the entity's component store. Always returns true. |

## Integration Patterns

### Standard Usage
This class is not invoked directly. It is configured in an NPC's behavior assets and executed by the server's behavior processing system. The following example illustrates how a hypothetical behavior system would use a pre-configured instance.

```java
// In a hypothetical NPC behavior processing loop:
// The 'action' instance is provided by the behavior tree/state machine.
// It was constructed from asset data prior to this execution.

ActionBase action = currentBehaviorNode.getAction(); // This would be an ActionSetAlarm instance

if (action instanceof ActionSetAlarm) {
    // The system executes the action, modifying the entity's state
    boolean success = action.execute(entityRef, currentRole, sensorInfo, deltaTime, worldStore);

    if (success) {
        // The action completed, so the behavior tree can transition to a new state.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionSetAlarm()`. The object is fundamentally data-driven and must be configured and created by its corresponding builder, BuilderActionSetAlarm, which is managed by the asset pipeline.
- **Stateful Use:** Do not cache and reuse an ActionSetAlarm instance across multiple ticks or for multiple NPCs. Each execution should correspond to a new, transient instance.
- **Off-Thread Execution:** Calling the `execute` method from any thread other than the main server tick thread will corrupt the EntityStore and lead to severe, unpredictable simulation errors.

## Data Pipeline
The flow of data for configuring and executing this action follows two distinct phases.

**Configuration-Time Flow:**
> NPC Behavior Asset (e.g., JSON file) -> Asset Loader -> **BuilderActionSetAlarm** -> **ActionSetAlarm Instance**

**Execution-Time Flow:**
> WorldTimeResource (Input) -> **ActionSetAlarm.execute()** -> EntityStore (Output: Writes to an entity's Alarm component)


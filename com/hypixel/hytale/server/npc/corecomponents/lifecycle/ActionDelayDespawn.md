---
description: Architectural reference for ActionDelayDespawn
---

# ActionDelayDespawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient Command

## Definition
```java
// Signature
public class ActionDelayDespawn extends ActionBase {
```

## Architecture & Concepts
The ActionDelayDespawn class is a concrete implementation of the **Command Pattern** within the server-side NPC artificial intelligence framework. As a subclass of ActionBase, it represents a single, atomic operation that can be scheduled and executed by an NPC's behavior controller, such as a Behavior Tree or a Finite State Machine.

Its sole responsibility is to manipulate the despawn timer of an NPCEntity. This provides a direct mechanism for an NPC's behavior logic to influence its own lifecycle. For example, an NPC might be configured to despawn 30 seconds after completing its primary task, or a guard NPC might have its despawn timer reset every time it spots a player.

The action's logic is controlled by two key parameters:
1.  **time:** The target despawn delay in seconds.
2.  **shorten:** A boolean flag that fundamentally alters the action's behavior.
    - If **true**, the action will only update the despawn timer if the new time is *less than* the current despawn time. This is an optimization pattern, ensuring that a series of actions cannot indefinitely postpone an NPC's cleanup.
    - If **false**, the action will unconditionally set the despawn timer to the new value, effectively acting as a reset.

This component acts as a bridge between the high-level AI decision-making layer (the Role) and the low-level entity lifecycle management system.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly. They are instantiated by the corresponding BuilderActionDelayDespawn, which is itself typically configured and created by a higher-level system that parses NPC role definitions from data files (e.g., JSON or HOCON).
-   **Scope:** An instance of ActionDelayDespawn is owned by the NPC's Role or behavior definition. It is stateless after creation and persists as long as the NPC's behavior configuration is loaded in memory.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is destroyed and its associated Role and behavior definitions are unloaded.

## Internal State & Concurrency
-   **State:** The internal state, consisting of the *time* and *shorten* fields, is **immutable**. It is set once during construction via the builder and is never modified thereafter.
-   **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The execute method modifies the state of an NPCEntity component. It is designed to be invoked exclusively from the server's main game thread during the designated AI update phase for a specific entity. Calling execute from other threads will lead to race conditions and world state corruption.

## API Surface
The public contract is limited to the execute method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Modifies the despawn timer of the target NPCEntity. Always returns true to signal successful completion to the AI scheduler. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly in procedural Java code. It is designed to be defined declaratively as part of an NPC's behavior configuration and executed by the AI system.

A conceptual data definition might look like this:

```yaml
# Example: Fictional NPC Role Definition
actions:
  - type: "DelayDespawn"
    time: 60.0
    shorten: false
```

The AI engine would then parse this definition, create an ActionDelayDespawn instance, and invoke its execute method at the appropriate time in the NPC's update loop.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionDelayDespawn()`. The object's internal state is uninitialized without its builder. This will lead to runtime exceptions or undefined behavior.
-   **Manual Invocation:** Do not call the execute method from outside the context of an NPC's Role-driven update tick. The method relies on a valid and current EntityStore reference and assumes it has exclusive write access to the NPCEntity component for that tick.

## Data Pipeline
This action serves as a command that injects data into the entity lifecycle system.

> Flow:
> AI Scheduler (e.g., Behavior Tree) -> **ActionDelayDespawn.execute()** -> Modifies NPCEntity.despawnTime -> EntityStore Update -> World Entity Manager -> Entity Despawn & Cleanup


---
description: Architectural reference for ActionResetInstructions
---

# ActionResetInstructions

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class ActionResetInstructions extends ActionBase {
```

## Architecture & Concepts
ActionResetInstructions is a concrete implementation of the `ActionBase` class, representing a single, discrete operation within the server-side NPC AI framework. Its primary function is to modify the instruction set of an NPC's `Role` component, which dictates the NPC's immediate, low-level tasks.

This class acts as a control mechanism for an NPC's behavior tree or finite state machine. It allows higher-level AI logic to clear or reset specific tasks. For example, if an NPC is patrolling and is suddenly alerted to a threat, this action can be invoked to clear the existing "walk to point X" instructions before new "attack target Y" instructions are issued.

The most critical architectural pattern employed is the use of a **deferred action**. The `execute` method does not perform the reset immediately. Instead, it enqueues a lambda onto the target `Role`'s deferred action queue. This design guarantees that modifications to the NPC's state occur at a safe, predictable point within the main AI processing tick, preventing state corruption and race conditions that could arise from modifying the instruction list while it is being executed.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's NPC asset loading pipeline. An instance is constructed from a `BuilderActionResetInstructions` object, which is itself populated by parsing a declarative asset file (e.g., JSON). This class is not intended for dynamic instantiation during gameplay.
- **Scope:** The object's lifetime is tied to the NPC's behavior definition. It is a stateless, reusable action held by a node in a behavior tree or a state in a state machine. It is invoked when its containing logic node becomes active.
- **Destruction:** Ownership is managed by the NPC's behavior definition. The object is eligible for garbage collection when the parent NPC entity is unloaded or its behavior asset is reloaded.

## Internal State & Concurrency
- **State:** This class is effectively **immutable**. Its internal `instructions` array is marked as final and is populated only once during construction. It holds no other mutable state, making it inherently a data-transfer object for a command.
- **Thread Safety:** The class itself is thread-safe due to its immutability. However, its operational context is strictly single-threaded. All interactions with the `Role` component, including the scheduling and execution of the deferred action, must occur on the primary server thread responsible for NPC updates. The deferred execution model is a key part of enforcing this single-threaded access pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Schedules a deferred task to reset NPC instructions. The actual reset operation is O(N), where N is the number of instructions to clear. Always returns true to indicate successful scheduling. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, procedural invocation by developers. It is designed to be configured within an NPC's asset files and executed automatically by the AI engine. The following example is conceptual, illustrating how the behavior engine would interact with the class.

```java
// Conceptual: How the AI engine uses the action
// An instance of this class is held by a behavior tree node.
ActionBase currentAction = behaviorTreeNode.getAction();

if (currentAction instanceof ActionResetInstructions) {
    // The engine calls execute, passing the current NPC context.
    currentAction.execute(npcRef, npcRole, npcSensorInfo, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance using `new`. The constructor requires builder objects that are only available during the asset loading process.
- **Bypassing Deferral:** Do not attempt to access and call the protected `resetInstructions` method directly. Doing so bypasses the deferred action queue, which can lead to instruction set corruption if called at an unsafe point in the AI tick.

## Data Pipeline
ActionResetInstructions facilitates a control flow rather than a data flow. It injects a command into the NPC's update cycle.

> Flow:
> AI Behavior Engine -> **ActionResetInstructions.execute()** -> Role Deferred Action Queue -> Role Update Phase -> **ActionResetInstructions.resetInstructions()** -> Role State Modification


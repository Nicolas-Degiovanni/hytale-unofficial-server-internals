---
description: Architectural reference for ActionReleaseTarget
---

# ActionReleaseTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class ActionReleaseTarget extends ActionBase {
```

## Architecture & Concepts
ActionReleaseTarget is a concrete command object within the server-side NPC Behavior System. It represents a single, atomic operation that an NPC can perform as part of a larger behavior, such as a Behavior Tree or a Finite State Machine.

Its sole function is to manipulate an NPC's internal memory by clearing a reference to a previously targeted entity. In the Hytale engine, NPCs can "mark" entities (players, other NPCs, items) and store them in numbered slots for later reference. This action instructs the NPC to forget the entity stored in a specific slot.

This class acts as a leaf node in a behavior graph. When the behavior system's traversal reaches this node, its execute method is called, the state change is applied instantly, and the action is considered complete.

## Lifecycle & Ownership
- **Creation:** Instances are not created dynamically during gameplay. They are instantiated once by the server's asset loading pipeline when parsing an NPC's behavior definition files (e.g., JSON assets). The corresponding builder, BuilderActionReleaseTarget, is responsible for its construction.
- **Scope:** The object's lifetime is tied to the loaded NPC definition. It is a shared, stateless template that persists as long as the NPC type is registered with the server. All NPC instances of the same type will share a single instance of this action.
- **Destruction:** The object is eligible for garbage collection only when its parent NPC definition is unloaded, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
- **State:** Immutable. The class holds a single final field, targetSlot, which is configured at creation time from the asset definition. It contains no per-NPC instance state, making it a pure, reusable command.
- **Thread Safety:** The class itself is inherently thread-safe due to its immutability. However, it operates on mutable state objects like Role and EntityStore. The calling system, typically the main NPC update loop, is responsible for ensuring that the execute method is not invoked concurrently for the same NPC, thereby preventing race conditions on the NPC's state.

**WARNING:** Do not call the execute method from an asynchronous task or a separate thread without external locking on the associated Role object. The NPC system expects actions to be executed sequentially within a single, designated update tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Clears the marked entity from the configured target slot on the NPC's Role. Always returns true, indicating the action completes successfully in a single tick. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay logic developers. It is automatically executed by the NPC's behavior processor. The processor traverses the NPC's active behavior tree and calls execute on this action when it becomes the active node.

A conceptual example of how the engine might use this:
```java
// Hypothetical code within an NPC Behavior Processor
// This code is conceptual and does not represent the actual engine implementation.

ActionBase currentAction = npcBehaviorTree.getActiveAction();

if (currentAction instanceof ActionReleaseTarget) {
    // The engine invokes the action, passing in the NPC's live state
    boolean success = currentAction.execute(entityRef, npcRole, sensorInfo, deltaTime, entityStore);
    if (success) {
        // The action is complete, so the behavior tree can move to the next state
        npcBehaviorTree.advance();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ActionReleaseTarget()`. The `targetSlot` and other base properties must be configured via the asset pipeline and its corresponding builder. Manual creation will result in an unconfigured and non-functional action.
- **Stateful Misuse:** Do not attempt to modify this class to hold runtime state. It is a shared definition object. All dynamic, per-NPC data must be stored within the Role object or other stateful components passed into the execute method.

## Data Pipeline
ActionReleaseTarget does not process a flow of data; rather, it is an endpoint in a control flow that triggers a state change.

> Control Flow:
> NPC Behavior Processor -> **ActionReleaseTarget.execute()** -> Role.getMarkedEntitySupport().clearMarkedEntity(slot) -> NPC internal state is modified.


---
description: Architectural reference for ActionSetFlag
---

# ActionSetFlag

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetFlag extends ActionBase {
```

## Architecture & Concepts
The ActionSetFlag is a fundamental command object within the server-side NPC AI framework. It functions as a "leaf node" in a Behavior Tree or a final state in a Finite State Machine. Its sole responsibility is to perform a single, atomic state change: setting a boolean flag on an NPC's `Role` component to a predefined value.

This component is not intended for complex logic. Instead, it serves as a primitive building block that enables more sophisticated behaviors. By setting flags, an NPC can record state changes—for example, "hasPatrolRouteCompleted" or "isHostile"—which can then be evaluated by other AI components, such as `Condition` nodes, to alter the flow of the behavior tree.

Instances of ActionSetFlag are configured at asset load time and are immutable, making them highly reusable across multiple NPC entities of the same type.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the NPC asset loading system during server bootstrap or when NPC definitions are loaded. The constructor signature, requiring a `BuilderActionSetFlag` and `BuilderSupport`, confirms that it is built from a data definition (e.g., a JSON file) rather than being created programmatically during gameplay.
- **Scope:** Session-scoped. An instance of ActionSetFlag persists for as long as its corresponding NPC asset definition is loaded in memory. The same instance is shared across all NPC entities that use this specific action definition.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated NPC assets, typically during a full server shutdown or a hot-reload event.

## Internal State & Concurrency
- **State:** Immutable. The internal fields `flagIndex` and `value` are marked as `final` and are initialized only once in the constructor. The object itself holds no runtime state specific to any single NPC entity.
- **Thread Safety:** The ActionSetFlag object is inherently thread-safe due to its immutable nature. However, the context in which it operates is not. The `execute` method modifies the state of a `Role` object, which is a live component of an NPC.

    **WARNING:** The engine's AI scheduler must guarantee that the `execute` method for any given NPC is only ever invoked from a single thread, typically the main server tick thread for the world region the NPC inhabits. Concurrent calls to `execute` on the same `Role` instance will lead to race conditions and undefined behavior.

## API Surface
The public contract is limited to the `execute` method inherited from `ActionBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Modifies the state of the provided `Role` by setting a flag at a pre-configured index. Always returns `true` to signify immediate and successful completion. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code by developers. It is designed to be defined within an NPC's data assets and invoked automatically by the AI behavior processor. The system uses this action as a primitive to change state.

A conceptual behavior tree runner would invoke the action like this:

```java
// Conceptual AI Runner (Engine Internals)
// This code is executed by the server's AI processing loop.

// Assume 'currentAction' is a reference to a pre-loaded ActionSetFlag instance
if (currentAction instanceof ActionSetFlag) {
    // The execute method is called, changing the NPC's internal state.
    boolean success = currentAction.execute(npcEntityRef, npcRole, npcSensorInfo, deltaTime, entityStore);

    // Because ActionSetFlag always returns true, the behavior tree
    // would immediately transition to the next state or action.
    if (success) {
        behaviorTree.moveToNextNode();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance using `new ActionSetFlag()`. The constructor's dependency on internal builder classes makes this impractical and bypasses the asset pipeline. All actions must be defined in data files.
- **Stateful Subclassing:** Do not extend this class to add runtime state. Actions are designed to be stateless, reusable commands. If you need to store data, use the NPC's `Role` or other components as the state container.
- **Ignoring Return Value Semantics:** The `true` return value has a specific meaning in the AI system: "this action completed instantly and successfully". Do not build logic that polls this action or expects it to return `false`.

## Data Pipeline
The data flow for this component is not driven by network I/O but by the internal AI decision loop.

> Flow:
> AI Behavior Tree Traversal → **ActionSetFlag.execute()** → Role.setFlag(index, value) → Modified `Role` State → Subsequent AI `Condition` Nodes Read Flag State


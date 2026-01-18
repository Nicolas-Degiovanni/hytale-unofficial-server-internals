---
description: Architectural reference for ActionFlockState
---

# ActionFlockState

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionFlockState extends ActionBase {
```

## Architecture & Concepts
ActionFlockState is a concrete implementation of the `ActionBase` contract, functioning as a stateless, single-purpose command within the server-side NPC AI framework. Its sole responsibility is to declaratively set the flocking state for an NPC.

This class acts as a bridge between the data-driven AI definitions (likely specified in HOCON or JSON asset files) and the live game state of an NPC. It is instantiated by the asset loading pipeline via its corresponding builder, `BuilderActionFlockState`. This pattern decouples the high-level AI behavior, such as a Behavior Tree or State Machine, from the low-level implementation of state management.

The action parses a state string, which can be hierarchical (e.g., "patrolling.suspicious"), into a primary state and an optional sub-state. When executed, it delegates the state change to the `StateSupport` component attached to the NPC's `Role`, effectively instructing the NPC to alter its group behavior logic.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's asset loading system through `BuilderActionFlockState` when an NPC's behavior definition is parsed. It is never created dynamically during the main game loop.
- **Scope:** Logically transient. An instance of ActionFlockState is typically owned by a parent node within a larger AI structure (e.g., a behavior tree task). It persists as long as that AI definition is loaded but holds no state between executions.
- **Destruction:** The object is garbage collected when the server unloads the corresponding NPC asset definitions or when an NPC's AI controller is replaced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** Immutable. The internal fields `state` and `subState` are final and are assigned only once during construction. This object carries no mutable state across multiple calls to its `execute` method.
- **Thread Safety:** Conditionally Safe. The object itself is immutable and can be safely referenced from any thread. However, the `execute` method is a mutating operation on the `Role` and `EntityStore` passed into it. Callers, typically the world's AI scheduler, **must ensure** that `execute` is invoked only from the main server thread for that NPC's world region to prevent race conditions and data corruption.

## API Surface
The public contract is minimal, consisting of the constructor (for internal builder use) and the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Immediately sets the flock state on the NPC's Role. Always returns true, indicating the action is atomic and completes successfully in a single tick. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by gameplay systems. It is a component of a larger AI definition and is executed by the AI scheduler. The system automatically constructs and wires this action based on asset files.

A conceptual representation within an AI asset might look like this:

```json
// Fictional asset definition
{
  "type": "sequence",
  "children": [
    { "type": "isHealthLow" },
    {
      "type": "actionFlockState",
      "state": "fleeing.scatter"
    }
  ]
}
```
The game engine parses this and invokes the `execute` method on the resulting ActionFlockState object when the sequence is triggered.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionFlockState()`. The state strings are sourced from data assets, and direct instantiation bypasses the asset pipeline, leading to behavior that is disconnected from the intended NPC design.
- **Stateful Subclassing:** Do not extend this class to add multi-tick or conditional logic. Actions are designed to be simple, atomic operations. For more complex, stateful behavior, create a new, more sophisticated `ActionBase` implementation.

## Data Pipeline
The primary flow is one of control, translating a data definition into a state change within the game world.

> Flow:
> NPC Asset File -> Asset Loader -> **BuilderActionFlockState** -> **ActionFlockState Instance** -> AI Scheduler Tick -> `execute()` -> `Role.getStateSupport().flockSetState()` -> NPC State Updated


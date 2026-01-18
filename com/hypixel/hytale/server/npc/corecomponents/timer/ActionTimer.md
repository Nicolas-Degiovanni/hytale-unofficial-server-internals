---
description: Architectural reference for ActionTimer
---

# ActionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionTimer extends ActionBase {
```

## Architecture & Concepts
The ActionTimer class is a concrete implementation of the **Command Pattern**, designed to operate within the server-side Non-Player Character (NPC) behavior system. Its primary role is to act as a stateless bridge between declarative NPC behavior assets (e.g., HOCON or JSON files) and the imperative, stateful `Timer` component.

Each ActionTimer instance encapsulates a single, specific manipulation to be performed on a `Timer` object, such as starting, stopping, pausing, or modifying its parameters. It is not the timer itself; rather, it is the *action* that controls the timer.

This component is instantiated by the NPC asset loading pipeline and integrated into an NPC's behavior tree or finite-state machine. When the NPC's logic dictates that a timer action should occur, the `execute` method of the corresponding ActionTimer instance is invoked by the behavior system. The ActionTimer then retrieves a reference to the target `Timer` object and performs the configured operation. This design decouples the high-level NPC behavior definition from the low-level implementation of time-based mechanics.

## Lifecycle & Ownership
- **Creation:** ActionTimer instances are never created directly. They are instantiated by the server's asset loading system when parsing an NPC's behavior definition. A specific builder, such as `BuilderActionTimerStart`, is first created from the asset file, which is then used to construct and configure the final ActionTimer object.
- **Scope:** An instance persists for the lifetime of the NPC's loaded behavior configuration. It is effectively a stateless, reusable command object that can be executed multiple times.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the parent NPC entity is unloaded or its behavior asset is reloaded with a new configuration. It does not manage any unmanaged resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The ActionTimer is effectively **immutable** after its construction. It holds configuration data like `rate` or `repeating` which are read from the NPC asset file. The true mutable state (e.g., the timer's current value, its running status) is held exclusively within the separate `Timer` object that this class operates upon. It is critical to understand that ActionTimer is a stateless command, not a stateful container.
- **Thread Safety:** This class is **not thread-safe** and must not be considered as such. It is designed to be executed exclusively by the main server thread that manages NPC updates. All interactions with the underlying `Timer` object are assumed to occur in a single-threaded context within the game loop tick.

## API Surface
The public contract is intentionally minimal, exposing only the `execute` method required by the `ActionBase` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Executes the configured timer action. Always returns true to signal successful execution to the calling behavior system. |

## Integration Patterns

### Standard Usage
Direct invocation of ActionTimer is not a standard developer task. The class is designed to be executed by a higher-level NPC behavior system, such as a behavior tree node or state machine. The system calls `execute` during the NPC's update tick.

```java
// Conceptual example of how the NPC Behavior System would use ActionTimer
// This code does not exist in a single place but represents the flow.

// During an NPC's update tick...
ActionBase currentAction = npcBehaviorTree.getCurrentAction();

if (currentAction instanceof ActionTimer) {
    // The system invokes the execute method, passing in the current world context.
    boolean success = currentAction.execute(entityRef, npcRole, sensorInfo, deltaTime, worldStore);
    // The system uses the 'true' return value to advance the behavior tree.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionTimer()`. The object's configuration is complex and must be handled by the asset loading pipeline and its corresponding builders. Direct instantiation will result in a non-functional object.
- **State Inspection:** Do not attempt to read state from an ActionTimer instance. It is a write-only command object. To check the status of a timer, you must retrieve a reference to the actual `Timer` component from the NPC's entity store.
- **Cross-thread Execution:** Calling the `execute` method from any thread other than the NPC's designated update thread is strictly forbidden and will lead to race conditions and world state corruption.

## Data Pipeline
ActionTimer acts as a control-flow component, translating a behavior signal into a state change on a different component.

> Flow:
> NPC Behavior Tree Node -> **ActionTimer.execute()** -> Target `Timer` Component -> `Timer.start()` / `Timer.pause()` / etc. -> `Timer` State is mutated


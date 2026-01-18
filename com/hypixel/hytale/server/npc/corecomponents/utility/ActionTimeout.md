---
description: Architectural reference for ActionTimeout
---

# ActionTimeout

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Component (Stateful Decorator)

## Definition
```java
// Signature
public class ActionTimeout extends ActionWithDelay implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The ActionTimeout class is a foundational component within the server-side NPC Behavior System. It functions as a **Decorator** for other Action components, wrapping a target Action to introduce a time-based execution constraint, commonly known as a cooldown or delay.

Its primary architectural purpose is to control the frequency of NPC behaviors without modifying the underlying behavior's logic. For example, it can prevent an NPC from repeatedly attempting to cast a spell or perform a special attack by enforcing a mandatory wait period between executions.

The component's behavior is dictated by two key configurations:
1.  **Delay Duration:** The length of the timeout, inherited from the ActionWithDelay base class.
2.  **Delay Timing:** A boolean flag, *delayAfter*, determines whether the timeout is applied *after* a successful execution (a classic cooldown) or *before* the next execution is allowed (a warm-up or charge time).

It integrates into the NPC's lifecycle by proxying all standard lifecycle events (spawned, loaded, teleported) to the wrapped Action, ensuring the child component remains synchronized with the parent NPC entity's state.

## Lifecycle & Ownership
-   **Creation:** ActionTimeout instances are not intended for direct instantiation via the *new* keyword. They are constructed exclusively by the NPC asset loading pipeline, specifically through a corresponding builder class (BuilderActionTimeout) and the BuilderSupport factory. This process occurs when the server parses an NPC's behavior tree definition from its asset file.
-   **Scope:** The lifetime of an ActionTimeout object is tightly coupled to the NPC Role it is a part of. It is created when the NPC's behavior set is initialized and persists as long as that behavior configuration is active for the NPC entity.
-   **Destruction:** The component is marked for garbage collection when its parent Role is unloaded or the owning NPCEntity is removed from the world. The *unloaded* and *removed* methods serve as the explicit signals for state cleanup, which are propagated to the wrapped Action.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Its primary state is the internal timer managed by its parent, ActionWithDelay. This timer's state (e.g., isDelaying, delayRemaining) is modified on nearly every game tick during the NPC's update cycle via calls to canExecute and execute. It also holds an immutable reference to the decorated Action.
-   **Thread Safety:** **This component is not thread-safe and must not be accessed from multiple threads.** All NPC behavior logic is designed to be executed serially on the main server thread for the world in which the NPC resides. Any concurrent access will lead to severe race conditions in the delay timer and unpredictable NPC behavior. No internal locking mechanisms are present.

## API Surface
The public API is primarily consumed by the NPC's internal Role and behavior tree schedulers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Evaluates if the action can run. Checks its own delay timer and then delegates the check to the wrapped Action. This is the primary gatekeeping method. |
| execute(...) | boolean | O(1) | Executes the wrapped Action and subsequently prepares the internal delay timer for the next cycle. Returns true to signify completion. |
| registerWithSupport(Role) | void | O(1) | An initialization hook called after construction. It registers the wrapped Action and sets the initial state of the delay timer. |
| loaded(Role) | void | O(1) | Lifecycle hook. Propagates the loaded event to the wrapped Action. |
| spawned(Role) | void | O(1) | Lifecycle hook. Propagates the spawned event to the wrapped Action. |
| unloaded(Role) | void | O(1) | Lifecycle hook. Propagates the unloaded event to the wrapped Action. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly in procedural Java code. It is designed to be configured declaratively within an NPC's behavior asset file (e.g., a JSON or HOCON file). The server's asset loader then constructs the object graph.

A conceptual asset definition might look like this:
```json
// Example NPC Behavior Definition
{
  "type": "ActionTimeout",
  "delay": 5.0,         // 5 second cooldown
  "delayAfter": true,   // Cooldown starts after the action runs
  "action": {
    "type": "ActionPlayAnimation",
    "animation": "special_attack"
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ActionTimeout()`. This bypasses the critical builder and factory system, resulting in a null or improperly configured component that will cause NullPointerExceptions at runtime.
-   **External State Management:** Do not call internal state methods like `prepareDelay` or `startDelay` from outside the class. The state of the timer is managed exclusively by the `canExecute`/`execute` contract. External manipulation will break the component's logic.
-   **Reusing Instances:** An ActionTimeout instance is stateful and bound to a single NPC Role. Do not attempt to share instances between different NPCs or Roles.

## Data Pipeline
ActionTimeout does not process a data stream but rather acts as a gate in a control flow. Its position in the NPC update loop is critical.

> **Control Flow:**
>
> NPC Behavior Scheduler Tick → `Role.update()` → **ActionTimeout.canExecute()** → (Returns true) → **ActionTimeout.execute()** → `WrappedAction.execute()` → Timer Reset


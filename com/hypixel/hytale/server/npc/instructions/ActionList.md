---
description: Architectural reference for ActionList
---

# ActionList

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Transient

## Definition
```java
// Signature
public class ActionList {
```

## Architecture & Concepts

The ActionList is a fundamental component of the server-side NPC Artificial Intelligence system. It functions as a container and execution controller for a sequence of one or more Action objects. Conceptually, it represents a single, composite instruction or a "behavior block" that an NPC can perform.

This class is not merely a list; it is a stateful sequencer that orchestrates the lifecycle and execution of its contained Actions. It operates in two primary modes, controlled by internal flags:

1.  **Blocking Mode (Sequential):** When `blocking` is true, the ActionList executes its Actions one at a time, in array order. It will not proceed to the next Action until the current one reports completion. This is used for behaviors that require a strict sequence, such as "walk to point A, then play animation B".

2.  **Non-Blocking Mode (Concurrent):** When `blocking` is false, the ActionList evaluates and executes all eligible Actions simultaneously within a single game tick. This is suitable for managing a set of concurrent states or reactions, such as "strafe while shooting".

The `atomic` flag further modifies the non-blocking mode. If `atomic` is true, the entire list is only considered executable if *all* contained Actions can be executed.

ActionList embodies the **Composite** design pattern, allowing a collection of Action objects to be treated as a single, unified Action by higher-level AI constructs like a Role or a behavior tree.

### Lifecycle & Ownership

-   **Creation:** An ActionList is instantiated with a pre-defined array of Action objects. This typically occurs during the loading and parsing of NPC behavior configurations, where a `Role` or a similar high-level behavior controller constructs ActionLists to represent its various states or abilities. The static `EMPTY_ACTION_LIST` provides a shared, immutable instance for no-op behaviors.

-   **Scope:** The lifetime of an ActionList is tightly coupled to its owning behavior component (e.g., a `Role`). It persists as long as the NPC's behavior profile is loaded and active. It is not a global or session-scoped object.

-   **Destruction:** The object is eligible for garbage collection when its owning `Role` is unloaded or the parent NPC entity is removed from the world. The `clearOnce` method provides a mechanism to reset its internal execution state (like `actionIndex`) without requiring re-instantiation, allowing a single behavior to be re-triggered.

## Internal State & Concurrency

-   **State:** This class is highly **mutable**. It maintains an internal cursor, `actionIndex`, to track its progress through the sequence in blocking mode. The `blocking` and `atomic` booleans are also mutable state that fundamentally alter the execution logic. The contained Action objects are themselves stateful.

-   **Thread Safety:** ActionList is **not thread-safe**. Its internal state, particularly `actionIndex`, is read from and written to during execution. Concurrent calls to `execute` or `canExecute` on the same instance from multiple threads will lead to race conditions and unpredictable AI behavior. All interactions with an ActionList instance must be confined to a single thread, which is typically the main server game loop responsible for ticking the NPC.

## API Surface

The public API is primarily concerned with execution control and lifecycle event propagation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlocking(boolean) | void | O(1) | Sets the execution mode to sequential. Must be called before execution begins. |
| setAtomic(boolean) | void | O(1) | Sets the execution condition for non-blocking mode. Must be called before execution. |
| canExecute(...) | boolean | O(N) / O(1) | Checks if the list's conditions for execution are met. Complexity is O(N) for non-blocking lists and O(1) for blocking lists. |
| execute(...) | boolean | O(N) / O(1) | Executes the currently active Action(s). Returns true if a blocking sequence has completed its full run. |
| hasCompletedRun() | boolean | O(1) | In blocking mode, checks if the sequence has finished and resets the internal index. |
| setContext(...) | void | O(N) | Propagates contextual information to all child Actions. Part of the initialization phase. |
| registerWithSupport(Role) | void | O(N) | Forwards dependency registration calls to all child Actions. Part of the initialization phase. |
| loaded(Role) | void | O(N) | Propagates the `loaded` lifecycle event to all child Actions. |
| spawned(Role) | void | O(N) | Propagates the `spawned` lifecycle event to all child Actions. |
| clearOnce() | void | O(N) | Resets the execution state of this list and all child Actions, preparing it for a new run. |

## Integration Patterns

### Standard Usage

An ActionList is almost never used in isolation. It is managed by a parent AI component which drives its execution within the server's game tick.

```java
// Pseudo-code within an NPC's Role or State class

// During initialization
Action[] myActions = ... // Load from config
ActionList patrolSequence = new ActionList(myActions);
patrolSequence.setBlocking(true);

// During the game tick update method
public void onTick(double dt, /*... other context ...*/) {
    if (patrolSequence.canExecute(context...)) {
        boolean hasFinished = patrolSequence.execute(context...);
        if (hasFinished) {
            // The entire sequence (e.g., walk to point A, then B, then C) is complete.
            // The Role can now decide to transition to a new state.
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **State Mutation During Execution:** Do not call `setBlocking` or `setAtomic` after the ActionList has begun executing. This can lead to an inconsistent and undefined internal state. These flags should be configured once upon initialization.

-   **Sharing Mutable Instances:** Avoid sharing a single, mutable ActionList instance across multiple NPC entities. The internal state like `actionIndex` is specific to one NPC's progress through a behavior. Doing so will cause NPCs to corrupt each other's AI state. If a behavior is shared, each NPC must have its own instance of the ActionList.

-   **Ignoring `canExecute`:** Calling `execute` without first checking `canExecute` can result in wasted computation and may violate the preconditions of the underlying Action, leading to errors or unexpected behavior.

## Data Pipeline

ActionList acts as a control-flow gatekeeper and dispatcher rather than a data-processing pipeline. It directs the flow of execution based on its configuration and state.

> Flow:
> NPC Role (`onTick`) -> **ActionList.canExecute()** -> (Evaluates one or all child `Action.canExecute()`) -> Returns boolean
>
> NPC Role (`onTick`) -> **ActionList.execute()** -> (Dispatches to one or all child `Action.execute()`) -> Child Actions modify NPC components via `ComponentAccessor` -> State change in `EntityStore`


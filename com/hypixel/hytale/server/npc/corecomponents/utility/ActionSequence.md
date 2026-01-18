---
description: Architectural reference for ActionSequence
---

# ActionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Composite Component

## Definition
```java
// Signature
public class ActionSequence extends ActionBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The ActionSequence is a fundamental structural component within the server-side NPC AI framework. It embodies the **Composite design pattern**, serving as a container that groups multiple, individual child actions into a single, sequentially-executed block.

Architecturally, it functions as a "branch" node within an NPC's behavior tree. Its primary purpose is to enable the creation of complex, multi-step behaviors from simple, reusable "leaf" actions (e.g., MoveToAction, WaitAction, PlayAnimationAction).

The system treats an ActionSequence identically to any other singular ActionBase derivative. This transparency allows AI schedulers, such as the Role component, to manage a complex series of operations without needing to know its internal composition. All execution logic and lifecycle events received by the ActionSequence are delegated directly to its internal ActionList, which manages the state and progression of the child actions.

## Lifecycle & Ownership
The lifecycle of an ActionSequence is tightly coupled to the NPC asset definition and the in-world NPC entity.

-   **Creation:** An ActionSequence is never instantiated directly via its constructor. It is created exclusively by the NPC asset loading pipeline, specifically through a BuilderActionSequence which parses the server-side NPC definition files (e.g., JSON). This process occurs when an NPC's behavior graph is being constructed in memory.
-   **Scope:** An instance of ActionSequence persists for the lifetime of its owning Role component. It is part of the in-memory representation of an NPC's static behavior definition.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is removed from the world and its associated Role component is unloaded. There are no explicit destruction or cleanup methods; its memory is managed by the JVM.

## Internal State & Concurrency
-   **State:** The ActionSequence object itself is effectively stateless and structurally immutable. Its core component, a final ActionList, is assigned at creation and never replaced. However, the contained ActionList is highly stateful, as it must track the currently executing action within the sequence. Therefore, the composite state of the ActionSequence is the state of its underlying ActionList.
-   **Thread Safety:** **Not thread-safe.** The entire NPC AI and behavior system is designed to be executed exclusively on the main server world thread. Invoking any method on an ActionSequence from an asynchronous task or a different thread is a critical error and will lead to state corruption, race conditions, and server instability. All interactions must be synchronized with the server tick.

## API Surface
The public API of ActionSequence is primarily concerned with fulfilling the ActionBase contract by forwarding calls to its internal ActionList.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the sequence is able to execute by querying the current child action. |
| execute(...) | boolean | O(1) | Executes the current active action in the sequence. Returns true if the sequence is still running. |
| loaded(Role) | void | O(N) | Forwards the *loaded* lifecycle event to all contained actions. |
| spawned(Role) | void | O(N) | Forwards the *spawned* lifecycle event to all contained actions. |
| unloaded(Role) | void | O(N) | Forwards the *unloaded* lifecycle event to all contained actions. |
| removed(Role) | void | O(N) | Forwards the *removed* lifecycle event to all contained actions. |

*Complexity N refers to the number of child actions within the sequence.*

## Integration Patterns

### Standard Usage
Direct interaction with an ActionSequence instance is not a standard development pattern. Instead, developers define sequences declaratively within an NPC's JSON asset file. The engine's AI scheduler then manages its execution.

The following conceptual example shows how the system would invoke the sequence, not how a developer would write procedural code.

```java
// This code is conceptual and resides within the AI scheduler (e.g., Role)

// Assume 'currentAction' is an ActionBase, which could be an ActionSequence
if (currentAction.canExecute(ref, this, sensorInfo, dt, store)) {
    boolean isStillRunning = currentAction.execute(ref, this, sensorInfo, dt, store);
    if (!isStillRunning) {
        // The sequence has completed, move to the next behavior
        advanceBehavior();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionSequence()`. The object's internal state and context are complex and must be wired by the `BuilderSupport` and asset loading systems. Manual creation will result in a non-functional component that will throw NullPointerExceptions.
-   **State Manipulation:** Do not attempt to access the internal ActionList to modify the sequence at runtime. The behavior graph is intended to be immutable after being loaded. Dynamic behaviors should be handled by higher-level constructs like state machines or schedulers.
-   **Asynchronous Execution:** Do not call `execute` or any other method from outside the main server thread. This will break the single-threaded assumptions of the AI system.

## Data Pipeline
ActionSequence is part of a control flow rather than a data processing pipeline. Its role is to direct the flow of execution between different actions based on their completion status.

> **Control Flow:**
> NPC Asset Definition (JSON) -> Asset Loader -> BuilderActionSequence -> **ActionSequence Instance** -> Role Scheduler -> `ActionSequence.execute()` -> `ActionList.execute()` -> Individual Child `Action.execute()`


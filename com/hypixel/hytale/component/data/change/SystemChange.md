---
description: Architectural reference for SystemChange
---

# SystemChange

**Package:** com.hypixel.hytale.component.data.change
**Type:** Transient Value Object

## Definition
```java
// Signature
public class SystemChange<ECS_TYPE> implements DataChange {
```

## Architecture & Concepts
The SystemChange class is a fundamental component of the engine's Entity-Component-System (ECS) state management. It is not a service or manager, but rather an immutable **Command Object** that represents a single, atomic change to the set of active systems in the ECS world.

By implementing the DataChange marker interface, SystemChange signals its role within a broader change-tracking and event-sourcing architecture. Its primary purpose is to decouple the *request* to modify the active systems from the *execution* of that modification. This allows high-level game logic to request structural changes to the game loop (e.g., adding a new physics system when a minigame starts) without needing direct access to the core ECS orchestrator. The orchestrator processes these change objects in a controlled, sequential manner, ensuring world state consistency.

This class encapsulates a single operation—typically an addition or removal—and the target ISystem instance, forming a self-contained instruction for the ECS runtime.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by game logic or state managers that need to dynamically alter the behavior of the ECS world. A typical use case is a state machine creating a SystemChange object when transitioning between game states (e.g., from *Lobby* to *In-Game*).
- **Scope:** Extremely short-lived. A SystemChange object is a transient message, designed to exist only for the duration of its journey from producer to consumer, almost always within a single engine tick.
- **Destruction:** The object becomes eligible for garbage collection immediately after it has been processed by the consuming system, such as a ComponentSystemManager. There is no mechanism or expectation for long-term storage.

## Internal State & Concurrency
- **State:** **Strictly Immutable**. All internal fields, *type* and *system*, are declared as final and are initialized exclusively at construction time. This design guarantees that a change event cannot be altered after its creation.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its complete immutability, a SystemChange instance can be safely created on any thread and passed to the main game loop thread for processing without requiring any locks, synchronization, or defensive copies. This makes it an ideal data structure for asynchronous game logic to communicate with the core engine.

## API Surface
The public contract is minimal, exposing only the data required to interpret the command.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ChangeType | O(1) | Returns the type of operation to be performed, such as ADD or REMOVE. |
| getSystem() | ISystem | O(1) | Returns the concrete system instance that is the target of the operation. |

## Integration Patterns

### Standard Usage
The canonical use of SystemChange is to create an instance and submit it to a central processing authority, such as an ECS world or manager, which queues and applies changes at a safe point in the game loop.

```java
// A game state manager wishes to activate a new UI rendering system.
UIRenderingSystem newSystem = new UIRenderingSystem(uiContext);
SystemChange<UIComponent> addSystemCmd = new SystemChange<>(ChangeType.ADD, newSystem);

// The command is submitted to the central ECS processor for deferred execution.
ecsWorld.submitChange(addSystemCmd);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain references to SystemChange objects beyond the current frame or transaction. They represent a point-in-time event, not persistent state. Storing them can lead to memory leaks and attempts to apply stale changes to the world.
- **Reusing Instances:** Do not attempt to reuse a SystemChange instance for a different operation. Always create a new object for each distinct change to ensure immutability and clarity.

## Data Pipeline
SystemChange acts as a data packet flowing from game logic to the core ECS engine. It ensures that modifications to the engine's core loop are orderly and predictable.

> Flow:
> Game State Manager -> `new SystemChange(ADD, ...)` -> ECS Change Queue -> **SystemChange** -> ECS Orchestrator -> ISystem added to active systems list


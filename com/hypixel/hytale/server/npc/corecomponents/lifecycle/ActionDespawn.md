---
description: Architectural reference for ActionDespawn
---

# ActionDespawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionDespawn extends ActionBase {
```

## Architecture & Concepts
The ActionDespawn class is a concrete command within the server-side NPC Artificial Intelligence framework. As a subclass of ActionBase, it represents a terminal operation or "leaf node" within an NPC's behavior tree or state machine. Its singular, focused responsibility is to initiate the removal of an NPC from the game world.

This action provides a critical distinction between two modes of removal:
1.  **Forced Despawn:** An immediate and unconditional removal of the entity from the world's EntityStore. This is a direct, synchronous operation that bypasses any cleanup animations or graceful shutdown logic.
2.  **Graceful Despawn:** A flagged removal. Instead of deleting the entity directly, it retrieves the NPCEntity component and calls its setToDespawn method. This signals to other game systems that the entity should be removed at the next appropriate opportunity, allowing for processes like death animations, loot drops, or fade-out effects to complete.

The ActionDespawn operates within a strict execution context provided by the AI engine, receiving a reference to the target entity, its role, and a handle to the authoritative EntityStore.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by its corresponding builder, BuilderActionDespawn. This builder is typically configured within an NPC's static definition files (e.g., JSON behavior graphs) and materialized by a factory when the server loads AI assets.
-   **Scope:** An ActionDespawn instance is stateless beyond its initial configuration. It is designed to be shared across all NPC instances of the same type. Its lifetime is tied to the lifetime of the loaded NPC behavior graph, persisting until the server unloads the relevant assets or shuts down.
-   **Destruction:** The object is managed by the Java garbage collector and is reclaimed when the behavior graph that owns it is no longer referenced.

## Internal State & Concurrency
-   **State:** The internal state is immutable. The `force` boolean is set once during construction via the builder and cannot be changed thereafter. This design ensures that the action's behavior is predictable and consistent for the lifetime of the object.
-   **Thread Safety:** This class is **not thread-safe**. It performs direct, unsynchronized mutations on the EntityStore and its components. It is fundamentally designed to be executed exclusively on the main server thread that manages the corresponding world's entity tick cycle. Calling the execute method from any other thread will lead to race conditions, data corruption, or ConcurrentModificationExceptions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Executes the despawn logic for the entity specified by `ref`. Returns true to signal successful completion to the behavior tree. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is a component that is declared within an NPC's behavior definition. The AI engine is responsible for traversing the behavior graph and invoking the execute method at the appropriate time.

A conceptual behavior definition might look like this:

```json
// Hypothetical NPC Behavior Definition
{
  "id": "ai.behavior.FleeAndDespawn",
  "root": {
    "type": "Sequence",
    "children": [
      { "type": "ActionRunAwayFromTarget" },
      {
        "type": "ActionDespawn",
        "force": false
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate this class directly. Its builder, BuilderActionDespawn, must be used, and typically only through declarative asset files.
-   **Asynchronous Execution:** Do not call the execute method from a separate thread, a future, or any context outside of the server's main entity update loop. This will break the server's threading model.
-   **State Caching:** Do not hold a reference to an ActionDespawn instance to call it multiple times for the same entity. The behavior tree is the sole authority for determining when an action should execute.

## Data Pipeline
ActionDespawn acts as a terminal sink in a data pipeline. It consumes an entity reference and produces a side-effect (entity removal) that alters the global state of the game world. It does not pass data to subsequent stages.

> Flow:
> AI Engine Tick -> Behavior Tree Evaluation -> **ActionDespawn.execute()** -> EntityStore Mutation -> World State Update


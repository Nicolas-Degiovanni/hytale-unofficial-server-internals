---
description: Architectural reference for StartObjectiveInteraction
---

# StartObjectiveInteraction

**Package:** com.hypixel.hytale.builtin.adventure.objectiveshop
**Type:** Transient / Command Object

## Definition
```java
// Signature
public class StartObjectiveInteraction extends ChoiceInteraction {
```

## Architecture & Concepts
The StartObjectiveInteraction class is a specialized implementation of the Command Pattern, designed to encapsulate a single, discrete server-side action: starting a game objective for a player. It serves as a data-driven bridge between a generic user interaction system (represented by the parent ChoiceInteraction) and the specific game logic of the ObjectivePlugin.

This class is not a long-lived service. Instead, it is a lightweight, serializable data object. Its primary role is to be defined within game assets (e.g., configuration files for NPC dialogue or shop menus), loaded by the server, and executed when a player triggers the corresponding choice. The static CODEC field is central to its design, enabling the engine to deserialize this object from configuration files, effectively allowing game designers to script game events without writing code.

When its run method is invoked by the server's interaction handler, it uses the provided context (PlayerRef, Store) to command the ObjectivePlugin to begin a new objective, identified by its internal objectiveId.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary ways:
    1. **Deserialization (Standard):** The static CODEC is used by the asset loading system to instantiate the object from game data files. This is the intended and most common creation path.
    2. **Programmatic (Rare):** Direct instantiation via `new StartObjectiveInteraction(objectiveId)` is possible for dynamic, code-driven interactions, but this is an exceptional use case.
- **Scope:** The object's lifetime is extremely short. It typically exists only for the duration of a single transaction—from the moment a player's choice is processed until the `run` method completes.
- **Destruction:** The object is eligible for garbage collection immediately after its `run` method is executed, as the interaction handling system will not retain a reference to it.

## Internal State & Concurrency
- **State:** The internal state, consisting solely of the objectiveId, is effectively immutable after construction. The `run` method is stateless and does not modify the object itself.
- **Thread Safety:** The object is inherently thread-safe due to its immutable nature. However, the execution of its `run` method is **not** thread-safe and carries strict concurrency constraints. It is designed to be called exclusively from the main world thread that has safe access to the EntityStore. Calling `run` from an asynchronous task or a different thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is minimal, focusing entirely on execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(store, ref, playerRef) | void | O(N) | Executes the interaction. Commands the ObjectivePlugin to start the configured objective for the specified player. Complexity depends on the objective's startup logic. |
| getObjectiveId() | String | O(1) | Returns the unique identifier of the objective to be started. |

## Integration Patterns

### Standard Usage
This class is intended to be defined within a data file and executed by the engine's interaction system. A developer should never need to call the `run` method directly. The system handles the lifecycle automatically.

The primary interaction is with the ObjectivePlugin, which is retrieved as a singleton to perform the core game logic.

```java
// This code is representative of how the ENGINE uses the class.
// A developer would typically define this in a data file, not code.

// 1. An interaction is triggered by a player.
// 2. The engine retrieves the pre-configured object.
StartObjectiveInteraction interaction = getInteractionFromChoice(playerChoice);

// 3. The engine provides the necessary context and executes the command.
interaction.run(world.getStore(), player.getRef(), player.getPlayerRef());
```

### Anti-Patterns (Do NOT do this)
- **Manual Execution:** Do not call the `run` method directly. It requires a very specific and valid `Store` and `PlayerRef` context that is only guaranteed to be correct when provided by the server's core interaction handler.
- **State Modification:** Do not attempt to modify the objectiveId field after construction using reflection. The object is treated as immutable and changing its state post-creation can lead to undefined behavior.
- **Asynchronous Invocation:** Never invoke the `run` method from a separate thread. All world state modifications, including starting objectives, must be synchronized with the main server tick.

## Data Pipeline
The flow of data and control for this class follows a clear, server-authoritative path from game configuration to world state modification.

> Flow:
> Game Asset (e.g., JSON/HOCON) -> Server Asset Loader -> **StartObjectiveInteraction** (in memory) -> Player Action Packet -> Interaction Handler -> `run()` method -> ObjectivePlugin -> World State Change


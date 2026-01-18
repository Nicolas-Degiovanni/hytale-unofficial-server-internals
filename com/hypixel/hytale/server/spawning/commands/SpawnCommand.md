---
description: Architectural reference for SpawnCommand
---

# SpawnCommand

**Package:** com.hypixel.hytale.server.spawning.commands
**Type:** Transient

## Definition
```java
// Signature
public class SpawnCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The SpawnCommand class serves as a command aggregator, grouping all spawning-related server commands under a single namespace: *spawning*. It does not implement any game logic itself; instead, it acts as a router, delegating execution to one of its registered sub-commands. This design adheres to the Command Pattern, providing a clean separation of concerns between the command invocation system and the underlying game logic.

By extending AbstractCommandCollection, SpawnCommand becomes a container in the server's command tree. Its primary role during initialization is to instantiate and register its children, such as EnableCommand and DisableCommand.

The nested sub-commands, like EnableCommand, extend AbstractWorldCommand. This is a critical architectural choice. The AbstractWorldCommand base class ensures that any command operating on world state receives a valid, non-null World context. This pattern prevents commands from executing in an invalid state (e.g., without a world loaded) and provides safe, direct access to world-specific components like WorldConfig and the EntityStore.

## Lifecycle & Ownership
-   **Creation:** A single instance of SpawnCommand is created by the server's command registration system during the server bootstrap sequence. The system discovers command classes and instantiates them to build the command hierarchy.
-   **Scope:** The SpawnCommand instance is a long-lived object, persisting for the entire server session. It is held within a central command registry. The execution context, however, including the CommandContext object, is transient and created for each individual command invocation.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The SpawnCommand instance is stateful, as it maintains a list of its registered sub-commands. This state is populated exclusively within its constructor and is immutable for the remainder of its lifecycle. The nested command classes (EnableCommand, DisableCommand) are entirely stateless, with all required context passed into their execute method.
-   **Thread Safety:** This class is not thread-safe. Command execution is managed by the server's main game loop and is expected to occur on a single, world-specific thread. The execute method directly mutates the WorldConfig object, which is a critical piece of shared world state. Any attempt to invoke a command from an external thread will lead to race conditions and world corruption. The framework guarantees serialized access.

## API Surface
The primary interface for this class is through the server console. The programmatic API is relevant only to the sub-commands.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | **(On Sub-Commands)** Modifies the NPC spawning flag within the target WorldConfig. Marks the configuration for persistence. |

## Integration Patterns

### Standard Usage
This class is designed to be used by server administrators or through automated scripts via the server console. It is not intended for direct programmatic use in game logic.

```sh
# Example console usage
# Enables NPC spawning in the current world
spawning enable

# Disables NPC spawning using the alias
sp disable
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpawnCommand()` in game logic. The command system is responsible for the lifecycle of all commands.
-   **Manual Execution:** Never call the `execute` method directly. The command framework is responsible for constructing the CommandContext and providing the correct World and Store references. Bypassing the framework will result in a corrupt or incomplete execution environment.

## Data Pipeline
The flow for this command begins with user input and ends with a persistent change to the world's configuration state.

> Flow:
> Console Input (`/spawning enable`) -> Server Command Parser -> **SpawnCommand** (Routing to sub-command) -> **EnableCommand.execute()** -> World.getWorldConfig() -> WorldConfig.setSpawningNPC(true) -> WorldConfig.markChanged() -> World Persistence System

---



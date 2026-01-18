---
description: Architectural reference for WorldConfigSetSpawnCommand
---

# WorldConfigSetSpawnCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient

## Definition
```java
// Signature
public class WorldConfigSetSpawnCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldConfigSetSpawnCommand is a server-side command responsible for modifying a world's global spawn location. It serves as a direct control mechanism for server administrators, bridging user input from the command line to the persistent state of the game universe.

Architecturally, this class is an implementation of the **Command Pattern**. It encapsulates a request to alter world state into a standalone object. The server's command system discovers, registers, and executes this command, decoupling the command invoker (the command dispatcher) from the receiver (the WorldConfig object).

This command's primary function is to instantiate a GlobalSpawnProvider with a specific Transform (position and rotation) and assign it to the active WorldConfig. It demonstrates a key contextual awareness:
1.  **Explicit Input:** It can parse explicit position and rotation coordinates provided as command arguments.
2.  **Implicit Context:** If arguments are omitted, it intelligently derives the spawn location from the command sender's entity. This requires querying the Entity-Component-System (ECS) for the sender's TransformComponent and HeadRotation, showcasing a tight integration between the command system and the core entity simulation.

## Lifecycle & Ownership
-   **Creation:** A single prototype instance of WorldConfigSetSpawnCommand is instantiated by the command registration system during server bootstrap. It is registered under the name "setspawn" within the "world config" command group.
-   **Scope:** The prototype instance is a singleton that persists for the entire server session. However, its execution is transient. The execute method is invoked for a single command request and completes within the same game tick. The object itself holds no state related to a specific execution.
-   **Destruction:** The prototype instance is de-referenced and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless regarding its execution. The internal fields, such as positionArg and rotationArg, are immutable configurations defining the command's argument structure. They are initialized once in the constructor and never change. All state required for execution (the world, the sender, the entity store) is passed into the execute method via parameters.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread. The execute method directly mutates the WorldConfig object without any locking mechanisms. This is safe under the assumption that all world-mutating operations are serialized onto a single game loop thread.

    **Warning:** Invoking the execute method from any thread other than the main server thread will lead to race conditions, world state corruption, and server instability.

## API Surface
The primary public contract is its `execute` method, which is invoked by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the command logic. Resolves position and rotation, updates the WorldConfig, and sends a confirmation message. Throws GeneralCommandException if a non-player console user fails to provide a position. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a user through the server console or in-game chat. The command system handles parsing, context creation, and invocation.

```
# Console or Chat Input
/world config setspawn
# OR
/world config setspawn 100.5 64.0 250.2
# OR
/world config setspawn ~10 ~-5 ~
```

The system automatically resolves the command, populates a CommandContext, and calls the execute method on the registered WorldConfigSetSpawnCommand instance.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldConfigSetSpawnCommand()`. The command is useless unless it is properly registered with the server's command dispatcher. The system manages its lifecycle.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context validation, which can lead to unpredictable behavior and server errors.
-   **Stateful Implementation:** Do not add mutable instance fields to this class to store state between executions. Commands are designed to be stateless; all necessary information should be derived from the parameters passed to the `execute` method.

## Data Pipeline
The flow of data for this command begins with user input and ends with a persistent change to the world's configuration.

> Flow:
> User Input (e.g., `/world config setspawn 0 100 0`) -> Server Network Layer -> Command Dispatcher -> Argument Parser -> **WorldConfigSetSpawnCommand.execute()** -> WorldConfig.setSpawnProvider() -> WorldConfig.markChanged() -> World Persistence System


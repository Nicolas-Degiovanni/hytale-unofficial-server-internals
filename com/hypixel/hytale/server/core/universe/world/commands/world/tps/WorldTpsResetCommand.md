---
description: Architectural reference for WorldTpsResetCommand
---

# WorldTpsResetCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world.tps
**Type:** Transient

## Definition
```java
// Signature
public class WorldTpsResetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldTpsResetCommand is a server-side command handler responsible for resetting a specific game world's simulation rate to a default value. It is a concrete implementation within the server's Command System framework.

Architecturally, this class serves as a stateless "verb" that acts upon a "noun" (the World object). By extending AbstractWorldCommand, it delegates the responsibility of context resolution to the command system. The framework ensures that when this command is executed, it is provided with a valid World instance to operate on, decoupling the command's logic from the complexities of world lookup and player context.

Its sole function is to modify a critical, mutable property of the World's game loop—the Ticks Per Second (TPS)—and provide feedback to the user who initiated the command. The default TPS value is hardcoded within the execute method, indicating a fixed, engine-defined standard.

### Lifecycle & Ownership
-   **Creation:** An instance of WorldTpsResetCommand is typically created once by the CommandSystem during server initialization. The system scans for command classes and registers them in a central dispatcher.
-   **Scope:** The registered instance persists for the entire server session. However, its role is that of a stateless handler; it holds no data between executions.
-   **Destruction:** The instance is destroyed when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no member fields and all required objects (CommandContext, World) are passed as method parameters during execution. The default TPS and millisecond values are method-local constants.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the operations it performs are not. The `world.setTps` method modifies the state of a shared World object. It is critical that the CommandSystem guarantees that all world-mutating commands are executed on the corresponding world's main simulation thread to prevent race conditions with the game loop.

## API Surface
The public API is defined by the contract of its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Sets the target World's TPS to 30 and sends a success message to the context's issuer. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly in code. It is invoked by the server's command parsing system in response to user input, such as a server administrator typing a command into the console.

```java
// This class is invoked by the framework, not by developers.
// Example of user interaction:
// > /world my_world_instance tps reset
// [Server]: World 'my_world_instance' TPS has been reset to 30 (33.33ms).
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldTpsResetCommand()`. The command system handles instantiation and registration. Bypassing the framework will lead to a non-functional command that is not registered to handle any input.
-   **Manual Execution:** Calling the `execute` method manually is a design violation. This bypasses permission checks, context validation, and thread safety guarantees normally provided by the CommandSystem.

## Data Pipeline
This component acts as a control-flow endpoint, not a data processing stage. It receives a command and triggers a state change in another system.

> Flow:
> User Input (`/world tps reset`) -> CommandSystem Parser -> **WorldTpsResetCommand.execute()** -> World.setTps() -> Game Loop Scheduler Update

> Feedback Flow:
> **WorldTpsResetCommand.execute()** -> Message Builder -> CommandContext.sendMessage() -> Network Subsystem -> Client Chat UI


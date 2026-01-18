---
description: Architectural reference for SelectChunkSectionCommand
---

# SelectChunkSectionCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SelectChunkSectionCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SelectChunkSectionCommand is a server-side command handler that integrates the core command system with the BuilderToolsPlugin. Its sole responsibility is to translate a player's command invocation into a world selection action.

Architecturally, this class acts as a simple adapter. It receives a high-level execution context from the command system, extracts the player's current position from the Entity-Component System (ECS) via the TransformComponent, performs the necessary calculations to determine the boundaries of the 32x32x32 chunk section the player is in, and then delegates the actual selection logic to the BuilderToolsPlugin by enqueuing a selection task.

This command is a terminal node in the command processing chain; it does not invoke other commands or manage complex state. It is a fire-and-forget operation that offloads its work to a specialized subsystem.

### Lifecycle & Ownership
- **Creation:** A single instance of SelectChunkSectionCommand is instantiated by the server's command registration system during server bootstrap. It is discovered and registered alongside all other built-in commands.
- **Scope:** The command object is a stateless singleton that persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected when the server shuts down and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields and all data required for its operation is provided through the arguments of the execute method. The logic is purely functional based on its inputs.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the execute method is designed to be called exclusively by the server's main thread, which manages the game state. The method interacts with the ECS Store, which is not thread-safe for arbitrary writes. The final operation, BuilderToolsPlugin.addToQueue, safely defers the world modification logic to the builder tools system, which manages its own concurrency or executes tasks on the main game tick.

## API Surface
The public API is minimal, consisting only of the constructor for instantiation by the framework and the overridden execute method which serves as the command's entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SelectChunkSectionCommand() | constructor | O(1) | Initializes the command definition, setting its name and required permission group. |
| execute(...) | protected void | O(1) | Calculates the current chunk section bounds and enqueues a selection task with the BuilderToolsPlugin. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically invoked by the server's command processing system when a player with the appropriate permissions executes the corresponding command in-game.

**Player Action (In-Game Chat):**
```
/selectchunksection
```

**System-Level Invocation (Conceptual):**
```java
// The server's CommandSystem finds the registered command and invokes it.
// This code is conceptual and does not represent user-facing code.
CommandSystem commandSystem = server.getCommandSystem();
Command targetCommand = commandSystem.findCommand("selectchunksection");
targetCommand.execute(playerCommandContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SelectChunkSectionCommand()` in game logic. The command system manages its lifecycle.
- **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, context validation, and the command execution pipeline. This can lead to unstable server state or security vulnerabilities.

## Data Pipeline
The flow of data for this command begins with player input and ends with a task being added to a processing queue.

> Flow:
> Player Chat Input -> Server Command Parser -> **SelectChunkSectionCommand.execute()** -> ECS Store (Read Player Position) -> Calculate Chunk Bounds -> **BuilderToolsPlugin.addToQueue()** -> Builder Tools Worker

---


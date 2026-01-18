---
description: Architectural reference for MemoriesLevelCommand
---

# MemoriesLevelCommand

**Package:** com.hypixel.hytale.builtin.adventure.memories.commands
**Type:** Transient Command Object

## Definition
```java
// Signature
public class MemoriesLevelCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The MemoriesLevelCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. By extending AbstractWorldCommand, it establishes itself as a server-side command that operates within the context of a specific game World.

Its primary architectural role is to serve as a simple, read-only bridge between a user-initiated action (typing a command) and the state managed by the MemoriesPlugin. The class itself contains no business logic; it delegates the core task of data retrieval to the MemoriesPlugin, which acts as the source of truth for the "memories level". This separation of concerns ensures that command definitions are lightweight and decoupled from the underlying game systems they interact with.

## Lifecycle & Ownership
- **Creation:** An instance of MemoriesLevelCommand is created by the server's CommandSystem during plugin initialization. The parent MemoriesPlugin is responsible for discovering and registering this command with the server.
- **Scope:** The object's lifetime is bound to the server's command registry. It persists as a shared, reusable instance for the entire duration the server is running and the MemoriesPlugin is active.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the MemoriesPlugin is unloaded, at which point it is removed from the command registry.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields that store or cache data between executions. All required context, such as the target World and the command issuer, is provided as arguments to the execute method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the command system guarantees that the execute method is invoked on the main server thread (the world tick loop). This is a critical design constraint to prevent race conditions when accessing game state, such as the World or the MemoriesPlugin.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MemoriesLevelCommand() | constructor | O(1) | Constructs the command object. Sets the command name to "level" and registers a translation key for its description. |
| execute(context, world, store) | protected void | O(1) | The command's entry point. Retrieves the memories level for the given world and sends a translated success message to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in code. It is automatically executed by the server's command handler when a player or the console issues the corresponding command. The primary developer interaction is registering the command within a plugin.

```java
// Hypothetical registration within a Plugin's initialization phase
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new MemoriesCommandGroup()); // Parent command

// The MemoriesCommandGroup would then register MemoriesLevelCommand as a subcommand
// User interaction:
// /memories level
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the execute method directly. The server's CommandSystem is responsible for constructing the correct CommandContext and ensuring execution on the proper thread. Bypassing the system can lead to state corruption and thread-safety violations.
- **Manual Instantiation:** Do not create instances using new MemoriesLevelCommand() for any purpose other than registering it with the CommandSystem during server or plugin startup. The command registry manages the object's lifecycle.

## Data Pipeline
The flow of data for this command is unidirectional, originating from user input and terminating as a message sent back to the user.

> Flow:
> User Input (`/memories level`) -> Network Layer -> Server Command Parser -> **MemoriesLevelCommand.execute()** -> MemoriesPlugin.getMemoriesLevel() -> World.getGameplayConfig() -> Return `level` (int) -> Message Builder -> CommandContext.sendMessage() -> Network Layer -> Client Chat UI


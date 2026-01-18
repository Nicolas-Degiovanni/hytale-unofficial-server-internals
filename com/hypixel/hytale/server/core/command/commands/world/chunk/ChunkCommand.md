---
description: Architectural reference for ChunkCommand
---

# ChunkCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ChunkCommand class is a foundational component of the server's command system, acting as a *Command Group* or *Namespace* rather than a command that performs a direct action. It follows the **Composite Design Pattern**, where it serves as a container for a collection of more specific, leaf-node commands related to server world chunks (e.g., ChunkInfoCommand, ChunkLoadCommand).

Its primary architectural role is to provide structure and organization. By grouping all chunk-related administrative functions under a single `/chunk` entry point, it simplifies the command hierarchy, enhances discoverability for server operators, and reduces namespace pollution in the global command registry.

When a command like `/chunk info` is executed, the central CommandManager first routes the request to the ChunkCommand instance. ChunkCommand then introspects the subsequent arguments and dispatches the execution to the appropriate registered sub-command, in this case, ChunkInfoCommand. It holds no execution logic itself; its sole responsibility is registration and delegation.

### Lifecycle & Ownership
-   **Creation:** A single instance of ChunkCommand is created by the server's central CommandManager during the bootstrap or initialization phase. This process typically involves scanning for classes that extend a command base class and instantiating them for registration.
-   **Scope:** The object is stateless and its instance persists for the entire lifetime of the server session. It is held as a reference within the server's primary command registry.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared. There is no explicit destruction or cleanup logic.

## Internal State & Concurrency
-   **State:** The ChunkCommand object is effectively **immutable** after its constructor completes. The list of sub-commands is populated once during instantiation and is not modified during runtime. It holds no mutable game state.
-   **Thread Safety:** This class is inherently thread-safe due to its immutable nature. However, the execution of commands dispatched by it is managed by the server's command system, which typically serializes all command processing onto the main server thread. This ensures that sub-commands can safely interact with world state without requiring explicit locking.

## API Surface
The public API of this class is defined by its parent, AbstractCommandCollection. The methods of interest are exclusively used within its constructor to build the command group. There is no public API intended for consumption by other systems after initialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkCommand() | Constructor | O(N) | Initializes the command group, setting its name and description. Registers N sub-commands. |

## Integration Patterns

### Standard Usage
Developers should not interact with an instance of this class directly. The standard pattern for extending its functionality is to create a new command class that implements the required logic and then register it within the ChunkCommand constructor.

```java
// To add a new sub-command, modify the ChunkCommand constructor:
public class ChunkCommand extends AbstractCommandCollection {
   public ChunkCommand() {
      super("chunk", "server.commands.chunk.desc");
      // ... existing sub-commands
      this.addSubCommand(new ChunkUnloadCommand());

      // Add your new command here
      this.addSubCommand(new MyNewChunkUtilityCommand());
   }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ChunkCommand()` in game logic. The object will be inert and un-registered, having no effect on the server's command system.
-   **Modifying Sub-Commands at Runtime:** Do not attempt to retrieve this object from the command registry to add or remove sub-commands after server initialization. The collection is not designed for runtime modification and doing so could lead to concurrency issues and unpredictable behavior.
-   **Bypassing the Dispatcher:** Do not attempt to get a reference to a sub-command (e.g., ChunkInfoCommand) and invoke its execution method directly. All command execution must flow through the central CommandManager to ensure permissions, context, and thread safety are correctly handled.

## Data Pipeline
The ChunkCommand acts as a routing point in the data flow of a user-initiated command. It does not transform data but directs it to the appropriate handler.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Parser -> CommandManager -> **ChunkCommand** (Dispatch) -> Sub-Command (e.g., ChunkInfoCommand) -> World State Access -> Command Result -> Sender (Player)


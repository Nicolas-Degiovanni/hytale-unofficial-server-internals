---
description: Architectural reference for ChunkLightingCommand
---

# ChunkLightingCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Singleton

## Definition
```java
// Signature
public class ChunkLightingCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkLightingCommand is a server-side diagnostic tool that exposes internal world lighting data to a privileged user, such as a server administrator or developer. It functions as a concrete implementation of the Command Pattern, integrating directly into the server's command processing system.

Architecturally, this class serves as a translation layer between a high-level user action (typing a command) and a low-level query against the world's data model. By extending AbstractWorldCommand, it inherits the necessary context and boilerplate for world-specific operations, ensuring that it executes within the correct server state and has access to the target World object.

Its primary responsibility is to:
1.  Define and parse a required 3D block position argument from the command input.
2.  Translate the block position into chunk and section coordinates.
3.  Query the World's ChunkStore to locate the specific BlockChunk and BlockSection.
4.  Access the lighting data, which is stored in an octree structure within the BlockSection.
5.  Serialize the lighting octree to a string and output it to the server logs.
6.  Provide user-facing feedback via the CommandContext, indicating success, failure, or if the target chunk is not currently loaded in memory.

## Lifecycle & Ownership
-   **Creation:** A single instance of ChunkLightingCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is discovered, instantiated via its default constructor, and stored in a central command registry.
-   **Scope:** The object's lifecycle is tied to the server session. It persists in the command registry for the entire duration the server is running.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the command registry is cleared, which typically occurs during a graceful server shutdown.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. The only member field, positionArg, is a final definition of the command's argument structure, configured once in the constructor. All state required for execution (the world, the command sender, the specific position) is passed into the execute method via the CommandContext.
-   **Thread Safety:** This class is **not thread-safe**. The execute method directly accesses and manipulates core world data structures like ChunkStore and BlockChunk. The server's architecture guarantees that all world-modifying commands are executed sequentially on the main server thread for the corresponding world.

    **WARNING:** Any attempt to invoke the execute method from an asynchronous task or a different thread will lead to severe concurrency issues, including data corruption and server instability. All interaction must be routed through the server's command dispatcher.

## API Surface
The public contract is exclusively defined by its role as a command. Direct programmatic invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the command logic. Complexity assumes the target chunk is already loaded in memory, making the lookup a constant-time hash map operation. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic use. It is automatically discovered and invoked by the server's command handler in response to user input from the console or an in-game client.

The conceptual invocation by the framework looks like this:

```java
// This is a conceptual example of framework-level invocation.
// Do NOT replicate this pattern in game logic.
CommandSystem commandSystem = server.getCommandSystem();
Command anAdminTyped = commandSystem.parse("/chunk lighting 150 64 -300");

// The system finds the ChunkLightingCommand instance and calls it
anAdminTyped.execute();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ChunkLightingCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in an object that is not registered with the server and is therefore useless.
-   **Manual Execution:** Never call the `execute` method directly. Bypassing the command system's dispatcher circumvents critical features like permission checks, argument validation, and thread safety guarantees. This is a high-risk action that can destabilize the server.
-   **Stateful Implementation:** Do not modify this class to hold state between executions. Commands must be stateless to ensure predictable behavior and prevent memory leaks.

## Data Pipeline
The flow of data for this command is unidirectional, originating from user input and terminating in the server logs.

> Flow:
> User Input (`/chunk lighting ...`) -> Server Network Layer -> Command Parser -> **ChunkLightingCommand.execute()** -> ChunkStore Lookup -> BlockSection Access -> LocalLight.octreeToString() -> HytaleLogger -> Server Log File & User Feedback Message


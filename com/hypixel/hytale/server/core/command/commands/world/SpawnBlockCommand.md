---
description: Architectural reference for SpawnBlockCommand
---

# SpawnBlockCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world
**Type:** Singleton Service

## Definition
```java
// Signature
public class SpawnBlockCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The SpawnBlockCommand class is a concrete implementation of the Command design pattern, tailored for server-side world manipulation. It resides within the server's command processing system and serves as the direct link between a user-initiated text command and a corresponding change in the game world's state.

Its primary architectural function is to encapsulate the logic for creating a new BlockEntity. It achieves this by:
1.  **Declarative Argument Parsing:** It defines its required arguments (block type, position, rotation) using the framework provided by its parent, AbstractWorldCommand. This separates the command's syntax and validation from its execution logic.
2.  **World State Mutation:** The core execute method directly interacts with high-level world objects like World and EntityStore to perform the entity creation.
3.  **Decoupling:** It decouples the command input source (e.g., a player's chat, a server console) from the action itself. The command dispatcher is responsible for routing the request to this class, which only needs a valid CommandContext to operate.

This class is a terminal node in the command processing pipeline, translating a validated request into a tangible in-game event.

### Lifecycle & Ownership
-   **Creation:** A single instance of SpawnBlockCommand is created by the server's central command registry during the server bootstrap or plugin loading phase. It is registered against the string identifier "spawnblock".
-   **Scope:** The instance is a long-lived singleton that persists for the entire server session. It does not hold per-execution state.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the command registry is cleared during server shutdown.

## Internal State & Concurrency
-   **State:** The class instance itself is effectively immutable after construction. Its fields (blockArg, positionArg, rotationArg) are final references to argument definition objects. The execute method is stateless; all required information for a single execution is passed in via the CommandContext, World, and Store parameters.

-   **Thread Safety:** This class is **not thread-safe** for execution. While its internal state is immutable, the execute method performs mutations on the World and EntityStore. The server's architecture mandates that all world-mutating commands are dispatched and executed on the main server thread to prevent catastrophic race conditions and ensure data consistency. Asynchronous invocation of the execute method will lead to world corruption.

## API Surface
The public contract is defined by the parent AbstractWorldCommand. The primary entry point is the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) Amortized | Parses arguments from the context, assembles a new BlockEntity, and adds it to the world's EntityStore. This is the core logic of the command. |

## Integration Patterns

### Standard Usage
This class is not designed to be invoked directly by developers. It is automatically discovered and managed by the server's command system. A user or system triggers its execution by issuing the corresponding text command.

The following example illustrates the conceptual flow within the command dispatcher, not code a developer would typically write.

```java
// Conceptual example of system-level invocation
CommandContext context = commandSystem.parse("/spawnblock hytale:stone ~ ~1 ~");
SpawnBlockCommand command = commandSystem.getCommand("spawnblock");
World primaryWorld = server.getWorld();

// The system ensures this is called on the main game thread
command.execute(context, primaryWorld, primaryWorld.getEntityStore().getStore());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new SpawnBlockCommand()`. The command will not be registered with the server and will be non-functional.
-   **Manual Invocation:** Calling the execute method manually bypasses the command system's critical pre-processing steps, including permission checks, context population, and thread safety guarantees. This can lead to security vulnerabilities and server instability.
-   **Asynchronous Execution:** Never call execute from a separate thread. All interactions with the World and its stores must be synchronized on the main server thread.

## Data Pipeline
The SpawnBlockCommand acts as the final processing stage for a specific data flow originating from a user command.

> Flow:
> Player Input (`/spawnblock ...`) -> Network Packet -> Server Command Dispatcher -> Argument Parser -> **SpawnBlockCommand.execute()** -> BlockEntity Factory -> EntityStore.addEntity -> World State Mutation -> Network Replication to Clients


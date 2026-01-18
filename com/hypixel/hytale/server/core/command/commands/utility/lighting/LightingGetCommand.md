---
description: Architectural reference for LightingGetCommand
---

# LightingGetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Command Handler

## Definition
```java
// Signature
public class LightingGetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The LightingGetCommand class is a server-side command handler responsible for querying and reporting the lighting data for a specific block coordinate within a game world. It serves as a concrete implementation within the server's broader Command System framework.

Architecturally, this class acts as a terminal node in the command processing pipeline. It translates a structured user request, parsed by the framework, into a direct query against the low-level world storage layer. Its primary responsibility is to interface with the World and its associated ChunkStore to retrieve raw lighting values, format them into a human-readable Message, and dispatch the result back to the command's originator via the CommandContext.

This command handler is specialized and has a narrow focus: it reads world state but does not modify it. It leverages the framework's declarative argument system (RequiredArg, FlagArg) to define its input contract, decoupling the command's logic from the complexities of parsing user input.

## Lifecycle & Ownership
- **Creation:** A single instance of LightingGetCommand is instantiated by the server's Command Registry during the server bootstrap sequence. It is registered under the *light get* command path.
- **Scope:** The object's lifecycle is tied to the server's runtime. It persists as a stateless handler within the Command Registry for the entire server session.
- **Destruction:** The instance is marked for garbage collection when the server shuts down and the Command Registry is cleared.

## Internal State & Concurrency
- **State:** The class instance itself is effectively immutable after construction. The fields defining the command's arguments (positionArg, hexFlag) are final and are configured once in the constructor. The execute method is purely procedural and does not modify any instance fields, ensuring that each command execution is independent and does not produce side effects on the handler object.

- **Thread Safety:** **This class is not thread-safe.** The execute method performs direct reads from the World and ChunkStore. These operations are fundamentally unsafe to perform outside of the main server thread. The command framework guarantees that execute is only ever invoked from the main server thread, preventing data races with the world simulation.

    **Warning:** Any attempt to invoke the execute method from an asynchronous task or a different thread will lead to world corruption, concurrency exceptions, or deadlocks.

## API Surface
The public API is exclusively consumed by the server's command processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(1) | Executes the command logic. Retrieves light data from the world at the position specified in the context. Sends the result or an error message back to the context's source. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is invoked automatically by the server's command system when a user or administrator executes the corresponding command in-game or via the server console.

The framework is responsible for creating the context and routing the call:

```java
// Conceptual representation of the framework's invocation
// A developer does NOT write this code.

// 1. User types: /light get 10 64 25 --hex
// 2. Framework parses input and finds the LightingGetCommand instance.
// 3. Framework populates CommandContext with parsed arguments.
CommandContext context = createPopulatedContext(...);
World world = server.getWorld("overworld");
Store<EntityStore> store = getEntityStoreForExecutor(...);

// 4. Framework invokes the handler
lightingGetCommandInstance.execute(context, world, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LightingGetCommand()`. The command must be managed by the server's Command Registry to function correctly. Direct instantiation creates an orphan object that is not registered to handle any commands.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the command system's parsing, permission checks, and context-building logic, which can lead to NullPointerExceptions or incorrect behavior.

## Data Pipeline
The flow of data for a typical execution of this command begins with user input and terminates with a message sent back to the user. The LightingGetCommand acts as the primary processor in the middle of this flow.

> Flow:
> User Command String -> Server Command Parser -> **LightingGetCommand** -> World::getChunkStore -> ChunkStore::getChunkReference -> BlockChunk::getSectionAtBlockY -> BlockSection::getGlobalLight -> ChunkLightData (Decode) -> Message (Format) -> CommandContext::sendMessage -> Network Packet -> User Console


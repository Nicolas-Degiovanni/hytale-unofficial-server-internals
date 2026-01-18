---
description: Architectural reference for BlockSetCommand
---

# BlockSetCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockSetCommand extends SimpleBlockCommand {
```

## Architecture & Concepts
The BlockSetCommand class is a concrete implementation of a server-side command, responsible for changing a single block within the game world. It operates within the server's Command System, a framework designed to parse and delegate text-based commands from players or the server console.

Architecturally, this class exemplifies the **Template Method Pattern**. It extends the abstract SimpleBlockCommand, which handles the boilerplate logic of parsing world coordinates (x, y, z) and resolving the corresponding WorldChunk. BlockSetCommand provides the specific implementation for the final step: validating the block type argument and applying the change to the world. This separation of concerns keeps the command logic clean, focused, and decoupled from the complexities of coordinate parsing and world lookups.

Its primary role is to act as a transactional endpoint that mutates the state of a WorldChunk based on validated user input.

### Lifecycle & Ownership
- **Creation:** A single instance of BlockSetCommand is created by the server's command registration system during the server bootstrap phase. The system scans for command implementations and instantiates them for its internal registry.
- **Scope:** The object is a stateless singleton for the duration of the server's runtime. It is designed to be instantiated once and reused for every execution of the corresponding command.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** and **stateless**. Its only instance field, blockArg, is a final object that defines the command's argument structure. This field is configured once in the constructor and never changes. All state related to a specific command execution (e.g., the sender, the target coordinates, the block type) is passed externally via the CommandContext parameter.

- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads without risk of race conditions or data corruption within the command object itself.

    **Warning:** While the command object is thread-safe, the underlying world systems it interacts with (specifically WorldChunk) may not be. The Command System is responsible for ensuring that world-mutating commands are executed on the main server thread or within a properly synchronized context to prevent world state corruption.

## API Surface
The primary public contract is its registration with the Command System, not direct method invocation. The core logic resides in a protected framework method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeWithBlock(context, chunk, x, y, z) | void | O(1) | Framework-internal method that performs the block mutation. Throws exceptions if context or chunk are null. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly. Interaction occurs by dispatching the command through the server's command processing pipeline, typically via player chat. The system handles instantiation and invocation.

The following example is a conceptual representation of how the framework invokes the command after parsing user input like `/set 10 64 20 stone`.

```java
// Conceptual framework invocation
CommandContext context = buildContextForPlayerInput("/set 10 64 20 stone");
CommandSystem commandSystem = server.getCommandSystem();

// The system finds the registered BlockSetCommand and executes it
commandSystem.dispatch(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockSetCommand()`. An instance created this way will not be registered with the server's Command System and will be completely non-functional. Commands must be managed by the framework.

- **Direct Method Invocation:** Never call `executeWithBlock` directly. Doing so bypasses critical upstream logic handled by the Command System and the SimpleBlockCommand parent, including:
    - Permission checks (e.g., Creative mode requirement).
    - Argument parsing and validation.
    - Coordinate resolution and chunk loading.
    - Error handling and user feedback.

## Data Pipeline
The BlockSetCommand acts as a final processing stage in the server's command handling pipeline. It translates a structured command context into a direct world state mutation.

> Flow:
> Player Input (`/set ...`) -> Network Packet -> Server Command Parser -> CommandContext Population -> **BlockSetCommand** -> WorldChunk.setBlock() -> World State Mutation -> (Optional) Network Sync to Clients


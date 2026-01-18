---
description: Architectural reference for BlockSetTickingCommand
---

# BlockSetTickingCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockSetTickingCommand extends SimpleBlockCommand {
```

## Architecture & Concepts
The BlockSetTickingCommand is a server-side command handler responsible for modifying a single block's ticking state within the game world. It is a concrete implementation within the server's Command System, designed to be discovered and managed by a central command registry.

This class extends SimpleBlockCommand, which abstracts the boilerplate logic of parsing block coordinates from the command arguments and locating the corresponding WorldChunk. BlockSetTickingCommand provides the specific implementation for the *setticking* action, directly manipulating the chunk data to enable the block's tick updates. It serves as a direct, user-invokable entry point for fine-grained world manipulation.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration service during the server bootstrap sequence. The system scans for command implementations and instantiates them for registration.
- **Scope:** The instance persists for the entire server session. Although the object itself is long-lived, its role is transient; it is stateless and only active during the execution of its specific command.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All required data for execution, such as the command sender and target coordinates, is provided via the CommandContext parameter in the execution method.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the operations it performs are not. The `executeWithBlock` method mutates the state of a WorldChunk.

    **WARNING:** All modifications to world state, including calls to `chunk.setTicking`, must be performed on the main server thread. The command system is responsible for ensuring this synchronization. Calling this command's logic from an asynchronous task will lead to chunk corruption and server instability.

## API Surface
The primary contract is fulfilled by overriding the `executeWithBlock` method from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeWithBlock(context, chunk, x, y, z) | void | O(1) | Enables the ticking state for the block at the specified coordinates within the given chunk. Sends a success message to the command sender. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is used implicitly by the server's command processing system. A user with appropriate permissions executes the command in-game or via the server console.

```
# Console or in-game chat
/setticking 100 64 -250
```

The command dispatcher routes this input to the registered BlockSetTickingCommand instance, which then executes its logic.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockSetTickingCommand()`. An instance created this way will not be registered with the server's command dispatcher and will be non-functional.
- **Manual Execution:** Avoid calling `executeWithBlock` directly. Doing so bypasses the critical coordinate parsing, chunk loading, and permission checks handled by the parent class and the command framework.

## Data Pipeline
The flow of data for this command begins with user input and ends with a direct modification to the world state.

> Flow:
> User Input (`/setticking ...`) -> Network Layer -> Command Parser -> **BlockSetTickingCommand** -> WorldChunk.setTicking -> World State Mutation


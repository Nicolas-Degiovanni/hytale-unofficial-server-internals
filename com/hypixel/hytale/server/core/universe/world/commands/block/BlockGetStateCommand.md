---
description: Architectural reference for BlockGetStateCommand
---

# BlockGetStateCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockGetStateCommand extends SimpleBlockCommand {
```

## Architecture & Concepts
The BlockGetStateCommand is a concrete implementation of the Command Pattern, designed for server-side diagnostics. It provides a mechanism for administrators and developers to inspect the live Entity-Component-System (ECS) state of a specific block within the game world.

Architecturally, this class leverages the Template Method Pattern through its parent, SimpleBlockCommand. The parent class is responsible for the boilerplate logic of a block-related command: parsing arguments, resolving block coordinates, and loading the corresponding WorldChunk. BlockGetStateCommand provides the specific implementation for the final step—inspecting the block and reporting its state—by overriding the `executeWithBlock` method.

Its primary function is to act as a read-only bridge between the server's command input system and the low-level world data storage. It queries the block's entity representation, retrieves its Archetype, and iterates through all attached Components to construct a human-readable state summary.

### Lifecycle & Ownership
- **Creation:** A single instance is created and registered by the server's central command management system during the server bootstrap sequence. It is not instantiated on a per-request basis.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is discarded when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no instance fields and its output is determined exclusively by the arguments passed to its execution method and the state of the world at that moment.
- **Thread Safety:** The class itself is immutable and therefore inherently thread-safe. However, the operations it performs are not. It reads from core world data structures like WorldChunk and ChunkStore. **WARNING:** Execution must be synchronized with the main server thread or world tick to prevent race conditions and data corruption. The command framework is responsible for enforcing this execution context.

## API Surface
The public contract is defined by the command framework via parent classes. The core logic resides in the protected `executeWithBlock` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeWithBlock(context, chunk, x, y, z) | void | O(C) | Retrieves and formats all Components (C) for the block entity at the specified coordinates. Sends the result to the CommandSender. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. It is automatically dispatched by the server's command processor in response to a player or console command.

```java
// Conceptual representation of the command framework invoking the command.
// A developer would NOT write this code.

// User types: /block getstate 10 64 20
String commandLine = "block getstate 10 64 20";

// The command system parses and dispatches to the registered instance.
Command command = commandRegistry.find("block getstate");
CommandContext context = createContextFor(sender, commandLine);
command.execute(context); // This eventually calls executeWithBlock internally.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockGetStateCommand()`. The command must be managed by the server's command registry to function correctly.
- **Bypassing the Framework:** Do not call `executeWithBlock` directly. This bypasses crucial pre-processing steps handled by the SimpleBlockCommand parent, such as argument parsing, coordinate validation, and safe chunk loading.

## Data Pipeline
The flow of data for a typical `getstate` command execution is linear, originating from user input and terminating with a message sent back to the user.

> Flow:
> User Command (`/block getstate ...`) -> Server Network Listener -> Command Parser -> **BlockGetStateCommand** -> WorldChunk::getBlockComponentEntity -> Archetype/Component Query -> String Builder -> Message System -> CommandSender -> Network Packet -> Client Console UI


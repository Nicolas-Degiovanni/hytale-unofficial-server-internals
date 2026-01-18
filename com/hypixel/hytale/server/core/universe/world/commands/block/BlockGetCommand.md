---
description: Architectural reference for BlockGetCommand
---

# BlockGetCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockGetCommand extends SimpleBlockCommand {
```

## Architecture & Concepts
The BlockGetCommand class is a specific, read-only command implementation within the server's Command System framework. It adheres to the Command Pattern, encapsulating the logic required to query the state of a single block in the game world.

Architecturally, it serves as a leaf node in the command hierarchy, inheriting from SimpleBlockCommand. This parent class abstracts away the boilerplate logic of parsing block coordinates and loading the corresponding WorldChunk. This design allows BlockGetCommand to focus exclusively on its core responsibility: retrieving block data and formatting a response.

This class acts as a bridge between three core systems:
1.  **Command System:** The entry point for user-initiated actions.
2.  **World State:** The source of truth for block data, accessed via the WorldChunk object.
3.  **Asset System:** Used to resolve a numerical block ID into a structured BlockType asset for richer metadata.

## Lifecycle & Ownership
-   **Creation:** A single instance of BlockGetCommand is typically instantiated by the CommandRegistry during server initialization or plugin loading. The registry scans for command definitions and registers them for runtime dispatch.
-   **Scope:** The instance persists for the entire server session, held as a reference within the CommandRegistry. It is stateless and reused for every invocation of the "get" block command.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the CommandRegistry is cleared, usually during server shutdown.

## Internal State & Concurrency
-   **State:** BlockGetCommand is **stateless**. It contains no mutable instance fields and does not cache data between calls. All required information is provided via the CommandContext and method parameters during execution.

-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, the overall safety of an operation depends on the underlying WorldChunk implementation. The command execution framework must guarantee that any modifications to the WorldChunk from other threads (e.g., world generation, physics updates) are properly synchronized to prevent inconsistent or partial reads.

    **Warning:** Concurrent execution of this command on the same WorldChunk that is being modified by another system can lead to data races if the WorldChunk is not thread-safe.

## API Surface
The public contract is primarily defined by the command framework, which invokes the protected implementation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockGetCommand() | constructor | O(1) | Constructs the command, registering its name ("get") and description key with the parent class. |
| executeWithBlock(context, chunk, x, y, z) | protected void | O(1) | Core logic. Queries the WorldChunk for block ID and support value, resolves the BlockType, and sends a formatted message to the CommandSender. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is designed to be discovered and executed by the server's command dispatch system in response to user input. A user, such as a player or server administrator, would invoke it via a chat or console command.

*Example User Input:*
```
/block get 100 64 -250
```

The system then dispatches this to the registered BlockGetCommand instance.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BlockGetCommand()` in application logic. The command must be registered with the central CommandRegistry to be discoverable and to participate in the framework's lifecycle, including permission checks and argument parsing.

-   **Direct Invocation:** Avoid calling `executeWithBlock` directly. Bypassing the command framework means you also bypass critical precursor logic handled by SimpleBlockCommand, such as coordinate validation and ensuring the target WorldChunk is loaded and accessible. This can lead to NullPointerExceptions or thread-safety violations.

## Data Pipeline
The flow of data for a typical `get` command execution is a linear, synchronous query from the user to the world state and back.

> Flow:
> User Input (`/block get...`) -> Network Layer -> Command Parser -> **BlockGetCommand** -> WorldChunk -> BlockType AssetMap -> Message Formatter -> CommandSender -> Network Layer -> Client UI


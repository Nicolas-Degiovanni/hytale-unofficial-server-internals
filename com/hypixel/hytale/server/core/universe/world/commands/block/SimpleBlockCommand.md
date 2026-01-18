---
description: Architectural reference for SimpleBlockCommand
---

# SimpleBlockCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class SimpleBlockCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
SimpleBlockCommand is a foundational component within the server's command framework, designed to streamline the creation of commands that operate on a single block within the world. It embodies the **Template Method Pattern**, providing a robust, asynchronous algorithm for resolving coordinates and loading world chunks, while deferring the final block-specific action to concrete subclasses.

The primary architectural purpose of this class is to enforce a safe and efficient pattern for world modification. By handling the complex, non-blocking chunk retrieval process, it abstracts away critical concurrency concerns from individual command implementations. This ensures that all block-related commands interact with the world data on the correct thread, preventing server lag, deadlocks, and race conditions. It serves as the essential bridge between the high-level Command System and the low-level, thread-sensitive World data layer.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of SimpleBlockCommand are instantiated once at server startup by the command registration system. They are not created on a per-execution basis.
- **Scope:** An instance of a command subclass persists for the entire server runtime, registered as a singleton within the central command dispatcher.
- **Destruction:** The object is garbage collected during server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding individual command executions. Its internal fields, such as the required coordinate argument definition, are configured at construction and are immutable thereafter. All transient data required for an operation is passed via the CommandContext.
- **Thread Safety:** This class is a cornerstone of the server's concurrency model for world interaction. The `execute` method can be safely called from any thread, as it immediately dispatches an asynchronous request to the world's dedicated thread pool. The subsequent call to the abstract `executeWithBlock` method is **guaranteed** by the `thenAcceptAsync(..., world)` continuation to execute on the correct world thread. This design is inherently thread-safe and is critical for server stability.

## API Surface
The public contract is defined by the methods that subclasses must interact with, primarily the abstract `executeWithBlock`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(async) | Framework entry point. Resolves coordinates, initiates a non-blocking chunk load, and schedules the final operation on the world thread. |
| executeWithBlock(context, chunk, x, y, z) | void | *Varies* | **Abstract template method.** Subclasses must implement this to perform the final block operation. This method is always executed on the correct world thread. |

## Integration Patterns

### Standard Usage
A developer never instantiates or calls SimpleBlockCommand directly. Instead, they extend it to create a new, specific command. The framework handles the invocation.

```java
// Example of a concrete command implementation
public final class SetStoneCommand extends SimpleBlockCommand {

    public SetStoneCommand() {
        super("setstone", "Sets the target block to stone.");
    }

    @Override
    protected void executeWithBlock(@Nonnull CommandContext context, @Nonnull WorldChunk chunk, int x, int y, int z) {
        // This logic is guaranteed to be on the correct world thread.
        // Direct mutation of the chunk is safe here.
        chunk.setBlock(x, y, z, BlockTypes.STONE);
        context.sender().sendMessage("Block set to stone.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking in executeWithBlock:** Never perform file I/O, network requests, or long-running computations within `executeWithBlock`. This method executes on a critical world thread; blocking it will cause severe server lag.
- **Manual Thread Management:** Do not attempt to bypass the asynchronous flow by manually managing threads or trying to access the WorldChunk outside of the `executeWithBlock` callback. The provided asynchronous pattern is the only safe way to interact with world data.
- **Incorrect Super Constructor:** Failing to call `super(name, description)` will result in a command that is not properly initialized or registered within the command system.

## Data Pipeline
The class manages a clear, asynchronous flow of data from user input to world state modification. This pipeline is designed to protect the main server loop from high-latency operations like chunk loading.

> Flow:
> Player Command String -> Command Parser -> **SimpleBlockCommand.execute()** -> Asynchronous Chunk Request to World -> World Thread Execution -> **SimpleBlockCommand.executeWithBlock()** -> Direct WorldChunk Data Mutation


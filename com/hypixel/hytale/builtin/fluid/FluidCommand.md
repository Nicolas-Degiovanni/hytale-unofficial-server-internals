---
description: Architectural reference for FluidCommand
---

# FluidCommand

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** Service Component

## Definition
```java
// Signature
public class FluidCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The FluidCommand class serves as a server-side administrative and debugging tool for direct manipulation of the world's fluid data. It is not a core game system but rather a high-level interface exposed through the server's command-line interface.

Architecturally, it follows the **Command Pattern**, encapsulating specific fluid operations into nested subcommand classes: GetCommand, SetCommand, and SetRadiusCommand. This class acts as a collection, registering these subcommands under the primary `/fluid` alias within the server's central command dispatcher.

The most critical design aspect of this system is its **asynchronous interaction with the world state**. All world modification operations are non-blocking. When a command needs to access or modify a block, it submits an asynchronous request to the World's ChunkStore for the relevant ChunkSection. The actual logic for reading or writing fluid data is provided as a callback lambda, which is executed by the world's dedicated thread pool only after the required chunk data has been loaded from disk or memory. This prevents server-wide stalls (lag) that would otherwise occur from synchronous I/O operations.

This command directly interacts with the low-level `FluidSection` component, a part of Hytale's Entity Component System (ECS) for world data representation.

## Lifecycle & Ownership
-   **Creation:** A single instance of FluidCommand is created by the server's command registration system during the server bootstrap sequence. Its constructor is responsible for instantiating and registering all of its subcommands.
-   **Scope:** The object is a stateless service component that persists for the entire lifetime of the server session.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and clears its command registry.

## Internal State & Concurrency
-   **State:** The FluidCommand class and its nested subcommand classes are **stateless**. All necessary context, such as the player executing the command and the parsed arguments, is passed into the `execute` method during each invocation. The class holds no mutable state related to the world.
-   **Thread Safety:** This class is inherently **thread-safe**. The `execute` methods are invoked by the command dispatcher, but they immediately delegate world interactions to the asynchronous `ChunkStore` API. The subsequent data mutations occur within callbacks scheduled on the world's main thread (`thenAcceptAsync(..., world)`), which guarantees safe, serialized access to chunk data and prevents race conditions.

**WARNING:** Any custom commands built on this pattern must strictly adhere to this asynchronous model. Directly modifying world components from any thread other than the one provided by the world's executor will lead to severe data corruption and server instability.

## API Surface
The primary contract of this class is its registration with the command system, not direct programmatic invocation. The `execute` method on its subcommands is the functional entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FluidCommand() | constructor | O(1) | Instantiates and registers the `get`, `set`, and `setradius` subcommands. |
| execute(...) | void | O(N³) (Async) | Defined in subcommands. Executes the fluid logic. Complexity is dependent on the operation; `setradius` is cubic in relation to the radius. All operations involve asynchronous I/O. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by other game systems. Its functionality is exposed exclusively through the server command line. A developer seeking to programmatically modify fluid data should use the underlying `ChunkStore` and `FluidSection` APIs directly, following the same asynchronous pattern.

Example of a plugin registering a similar, custom command:
```java
// In a plugin's initialization routine
// This demonstrates the pattern, not direct use of FluidCommand

// 1. Get the server's command registry
CommandRegistry registry = server.getCommandRegistry();

// 2. Create and register a new command collection
registry.register(new MyCustomFluidCommands());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Calling `new FluidCommand()` in your own code is useless. The object has no effect unless it is registered with the server's central command registry during startup.
-   **Blocking on Futures:** The `getChunkSectionReferenceAsync` method returns a `CompletableFuture`. Never call `.join()` or `.get()` on this future from the main server thread. Doing so defeats the purpose of the asynchronous design and will cause the entire server to freeze until the chunk is loaded.
-   **Cross-thread World Modification:** Do not modify the `FluidSection` or any other world component from a custom thread. All world state mutations must be scheduled back onto the `World`'s dedicated executor, as demonstrated by the use of `thenAcceptAsync(..., world)`.

## Data Pipeline
The flow of data for a command execution is asynchronous and thread-aware, ensuring server performance is maintained.

> Flow:
> Player Command Input -> Server Command Parser -> **FluidCommand** Subclass -> ChunkStore (Async Request) -> I/O Thread (Chunk Loading) -> World Thread Pool (Callback Execution) -> FluidSection Mutation -> WorldChunk Marked for Saving

---



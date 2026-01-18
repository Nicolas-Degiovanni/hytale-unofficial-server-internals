---
description: Architectural reference for WorldAddCommand
---

# WorldAddCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldAddCommand extends CommandBase {
```

## Architecture & Concepts
The WorldAddCommand class serves as a user-facing entry point for the complex, asynchronous process of world creation. It functions as a translator, converting a user-initiated command string into a direct, validated call against the core Universe service.

This class is a component of the server's command system. Its primary responsibility is not to contain the logic for world creation itself, but to:
1.  Define the command's syntax, including required and optional arguments (world name, generator, storage type).
2.  Parse and retrieve argument values from the CommandContext provided by the command processor.
3.  Perform initial, synchronous validation to prevent invalid requests from reaching the core system. This includes checking for name collisions with existing worlds, both in-memory and on-disk.
4.  Validate the specified world generator type against the server's registered IWorldGenProvider codecs.
5.  Delegate the core world creation task to the Universe singleton, which orchestrates the asynchronous operation.
6.  Manage the asynchronous lifecycle of the request, providing feedback to the CommandSender upon success or failure.

It acts as a crucial boundary layer, protecting the Universe service from malformed user input and abstracting the asynchronous nature of world generation into a simple, user-executable command.

### Lifecycle & Ownership
-   **Creation:** An instance of WorldAddCommand is created and registered by the server's command management system during server initialization. It is not instantiated on a per-request basis.
-   **Scope:** The singleton instance persists for the entire server session. Its execution logic, however, is invoked within the transient scope of a single command execution.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The object's state consists of its argument definitions (nameArg, genArg, storageArg). This state is configured once in the constructor and is **immutable** for the lifetime of the object. The class is effectively stateless from an execution perspective, as all per-request data is passed via the CommandContext parameter.

-   **Thread Safety:** This class is thread-safe. The `executeSync` method is designed to be called on the main server thread. It performs lightweight, non-blocking validation and then immediately delegates the I/O-intensive world creation task to the Universe service, which returns a CompletableFuture. This design prevents the command from blocking the main server tick loop. Callbacks for success and failure are handled via the CompletableFuture API, ensuring that feedback is delivered to the user in a thread-safe manner.

## API Surface
The public contract is defined by its inheritance from CommandBase and its constructor. The primary interaction point is the `executeSync` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldAddCommand() | constructor | O(1) | Initializes the command definition and its arguments. |
| executeSync(context) | void | O(N) | Executes the command logic. Complexity is O(N) where N is the number of loaded worlds, due to name collision checks. The subsequent world creation is asynchronous. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. It is designed to be discovered and executed by the server's command processing system in response to user input from the console or in-game chat.

The typical invocation is via a text command:
```
/world add <name> [gen:<generator_type>] [storage:<storage_type>]
```
Example:
```
/world add my_new_world gen:default_plus
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldAddCommand()` and call its methods. This bypasses the command registry, argument parsing, and permission checks. For programmatic world creation, interact with the Universe API directly.
    ```java
    // DO NOT DO THIS
    // This is fundamentally incorrect and bypasses the entire command system.
    WorldAddCommand cmd = new WorldAddCommand();
    // cmd.executeSync(...); // Fails to replicate a real CommandContext
    ```

-   **Programmatic Control Flow:** Do not use this command to create worlds from other plugins or server systems. It is designed for user interaction only. The correct approach is to acquire the Universe service and call its API.
    ```java
    // CORRECT programmatic approach
    Universe.get().addWorld("my_plugin_world", "default", "default");
    ```

## Data Pipeline
The flow of data for this command begins with user input and terminates with either a new world on disk or an error message sent back to the user.

> Flow:
> User Command String -> Server Command Parser -> CommandContext -> **WorldAddCommand.executeSync** -> Universe.addWorld (async) -> IWorldGenProvider -> Chunk Storage System -> Filesystem I/O
>
> Feedback Flow:
> CompletableFuture -> `thenRun` or `exceptionally` Callback -> CommandSender -> Message -> Network Packet -> Client UI


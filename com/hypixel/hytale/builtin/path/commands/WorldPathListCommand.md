---
description: Architectural reference for WorldPathListCommand
---

# WorldPathListCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class WorldPathListCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPathListCommand is a server-side, user-invokable command responsible for querying and displaying a list of all configured WorldPaths within a given World. As a subclass of AbstractWorldCommand, it is designed to operate specifically within the context of a game world, ensuring that its logic has safe access to world-specific data.

This class serves as a read-only interface between a command issuer (like a player or server console) and the internal pathing configuration of a World. It does not modify world state. Its primary role is to retrieve data from the WorldPathConfig, format it for human consumption, and send it back to the user via the CommandContext. It is a terminal component in the command execution chain, translating a user request into a formatted data response.

## Lifecycle & Ownership
-   **Creation:** A single instance of WorldPathListCommand is typically created by the server's command registration system during the server bootstrap phase. The system discovers command classes and instantiates them to build a central command registry.
-   **Scope:** The command object instance persists for the entire server session, held statically by the command registry. However, the execution context, including the CommandContext and World references, is ephemeral and scoped only to a single invocation of the execute method.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is unloaded, at which point the command registry is cleared.

## Internal State & Concurrency
-   **State:** WorldPathListCommand is stateless. It contains no mutable fields and all data required for its operation is provided as arguments to the execute method. The command name and description are immutable strings set during construction.
-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. However, the execution of its logic is not guaranteed to be safe if invoked from multiple threads simultaneously. The server's command system is responsible for ensuring that commands are executed on the main world thread, preventing race conditions or concurrent modification exceptions when accessing the World object's state. This class performs no internal locking.

## API Surface
The public contract is fulfilled by the command system's invocation of the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N log N) | Retrieves all WorldPath objects from the world, sorts them by name, and sends each name as a raw message to the command issuer. N is the number of paths. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is automatically discovered and invoked by the server's command handling system in response to user input. A server administrator or player would trigger its execution by typing the corresponding command in-game or in the server console.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldPathListCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose and bypasses registration.
-   **Manual Execution:** Do not call the `execute` method directly. This bypasses the server's permission checks, context setup, and thread safety guarantees, which can lead to inconsistent state or server instability.

## Data Pipeline
The data flow for a typical invocation is unidirectional, from the world's internal state to the user's client.

> Flow:
> User Input (`/worldpath list`) -> Server Command Parser -> **WorldPathListCommand.execute()** -> World.getWorldPathConfig() -> Collection of WorldPath objects -> Stream Sort & Map -> CommandContext.sendMessage() -> Network Message -> Client Chat UI


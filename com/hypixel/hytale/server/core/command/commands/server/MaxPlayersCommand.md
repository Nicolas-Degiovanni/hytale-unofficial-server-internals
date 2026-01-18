---
description: Architectural reference for MaxPlayersCommand
---

# MaxPlayersCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server
**Type:** Singleton

## Definition
```java
// Signature
public class MaxPlayersCommand extends CommandBase {
```

## Architecture & Concepts
The **MaxPlayersCommand** is a concrete implementation within the server's Command System framework. It encapsulates the logic for a single, user-facing server command: *maxplayers*. Its primary architectural role is to act as a bridge between parsed user input and the core server configuration state.

This class adheres to the Command pattern, inheriting from the abstract **CommandBase** which provides the necessary lifecycle hooks and integration points with the command dispatcher. It is responsible for two distinct operations:
1.  **Querying:** Retrieving the current maximum player limit from the server configuration.
2.  **Mutation:** Setting a new maximum player limit in the server configuration.

It directly interacts with the global **HytaleServer** singleton to access and modify the server's runtime configuration, demonstrating a tight coupling with the core server state management. Feedback to the command issuer is managed through the **CommandContext** object, abstracting away the details of the command source (e.g., console, in-game player).

## Lifecycle & Ownership
-   **Creation:** A single instance of **MaxPlayersCommand** is instantiated by the server's central **CommandRegistry** during the server bootstrap sequence. The registry is responsible for discovering and initializing all available command classes.
-   **Scope:** The object is a long-lived singleton. Its lifetime is tied directly to the server session, persisting from server start to server shutdown. It is held as a reference within the **CommandRegistry**.
-   **Destruction:** The instance is eligible for garbage collection only when the **CommandRegistry** is cleared during the server shutdown process. No explicit cleanup logic is required.

## Internal State & Concurrency
-   **State:** The class is effectively stateless with respect to its execution. It contains a final field, **amountArg**, which defines the command's argument structure. This field is initialized once in the constructor and is immutable thereafter. All state required for execution, such as the command sender and provided arguments, is passed via the **CommandContext** parameter.
-   **Thread Safety:** This class is not designed for concurrent execution. The method name **executeSync** is a strong convention indicating that the command system guarantees its execution on the main server thread. This design avoids the need for explicit locking or synchronization when accessing core server components like **HytaleServer.getConfig()**, which are generally not thread-safe.

**WARNING:** Invoking methods on this class from any thread other than the main server thread will lead to race conditions and unpredictable server state corruption.

## API Surface
The public contract is primarily defined by its superclass, **CommandBase**. The key implementation is the overridden **executeSync** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the command logic. Reads or writes the max players setting based on the provided arguments in the context. Sends a response message to the command issuer. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly by developers. It is invoked automatically by the server's command processing system. A user or system triggers it by providing a command string.

*Example user interaction:*
```
> /maxplayers 60
[Server] Maximum players set to 60.

> /maxplayers
[Server] Maximum players is 60.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MaxPlayersCommand()`. The command will not be registered with the server and will have no effect. The server's **CommandRegistry** is the sole owner of command instances.
-   **Direct Invocation:** Avoid calling the **executeSync** method directly. Doing so bypasses the server's command dispatch pipeline, which may include critical cross-cutting concerns like permission checks, logging, and error handling.

## Data Pipeline
The flow of data for a command invocation follows a clear, sequential path from user input to state mutation and feedback.

> Flow:
> User Input String (`/maxplayers 60`) -> Server Command Parser -> Command Registry Dispatcher -> **MaxPlayersCommand.executeSync** -> HytaleServer.getConfig().setMaxPlayers(60) -> CommandContext.sendMessage() -> Network Layer -> Client UI


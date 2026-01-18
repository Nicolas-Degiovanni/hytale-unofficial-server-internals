---
description: Architectural reference for NotifyCommand
---

# NotifyCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Transient

## Definition
```java
// Signature
public class NotifyCommand extends CommandBase {
```

## Architecture & Concepts
The NotifyCommand class is a concrete implementation of the Command Pattern, designed to integrate seamlessly with the server's central command processing system. Its primary function is to serve as a bridge between a privileged user's text input and the server-wide notification system.

Architecturally, it is a leaf node in the server's logic tree. It receives a pre-parsed context object, performs its specific business logic—argument parsing and validation—and then delegates the core task of broadcasting the message to a dedicated utility, NotificationUtil. This separation of concerns ensures that NotifyCommand is responsible only for command-specific parsing, not the underlying mechanics of network packet creation or player session management.

The class is responsible for interpreting a flexible argument structure, allowing the command sender to specify a NotificationStyle and a message payload. It supports both raw string messages and structured JSON messages, providing robust error handling for malformed input.

## Lifecycle & Ownership
- **Creation:** A single instance of NotifyCommand is created by the CommandSystem during server bootstrap. The system scans for classes extending CommandBase and instantiates them for registration.
- **Scope:** The object instance persists for the entire server session, held within the central command registry. It is designed to be reused for every invocation of the "notify" command.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** NotifyCommand is effectively stateless and immutable after construction. Its initial properties, such as the command name and description, are set once in the constructor. All state required for execution, such as the sender and input arguments, is provided via the transient CommandContext object passed to the executeSync method.

- **Thread Safety:** The class itself contains no mutable state, making the object instance inherently thread-safe. However, the execution contract is strict. The method name *executeSync* is a strong convention indicating that it **must** be invoked on the main server thread. The command system guarantees this contract. Any attempt to invoke this method from an asynchronous worker thread would violate the server's threading model and likely lead to race conditions or crashes when interacting with other game systems.

## API Surface
The public contract is minimal, consisting of the constructor for framework instantiation and the overridden execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N) | Parses the raw input string from the context to extract a notification style and message. Delegates to NotificationUtil to broadcast the message to all players. Complexity is linear based on the length (N) of the input arguments. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly via code. It is invoked by the server's command handler in response to a player or console input.

The user-facing interaction is:
```
/notify [STYLE] {message}
```
Example:
```
/notify URGENT {"text":"Server restarting in 5 minutes!"}
```

The framework's internal invocation looks like this:
```java
// Pseudo-code for CommandSystem invocation
CommandContext context = buildContextForPlayer(player, "/notify URGENT Server is restarting");
CommandBase command = commandRegistry.get("notify");

if (command != null) {
    // The CommandSystem ensures this is called on the main server thread
    command.executeSync(context);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never manually create an instance of NotifyCommand and call executeSync. This bypasses critical framework services like permission checking, context building, and thread safety guarantees provided by the CommandSystem.

    ```java
    // DO NOT DO THIS
    NotifyCommand cmd = new NotifyCommand();
    // This context is incomplete and the call is not thread-safe
    cmd.executeSync(new CommandContext(...));
    ```

- **Asynchronous Execution:** Do not wrap a call to executeSync in a separate thread or future. The method is not designed for concurrent execution and expects to be on the main server tick.

## Data Pipeline
The flow of data for a successful notification begins with user input and terminates with a UI update on all connected clients. NotifyCommand is a critical processing step in this pipeline.

> Flow:
> Player Chat Input -> Server Network Listener -> CommandSystem Parser -> **NotifyCommand.executeSync** -> NotificationUtil -> Player Session Manager -> Network Packet Broadcaster -> All Game Clients -> Client UI Notification Renderer


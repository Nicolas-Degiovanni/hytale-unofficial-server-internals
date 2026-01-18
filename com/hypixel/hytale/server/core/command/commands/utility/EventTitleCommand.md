---
description: Architectural reference for EventTitleCommand
---

# EventTitleCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Handler

## Definition
```java
// Signature
public class EventTitleCommand extends CommandBase {
```

## Architecture & Concepts
The EventTitleCommand is a concrete implementation of the Command Pattern, designed to be registered and managed by the server's central CommandSystem. Its primary architectural role is to serve as a high-level interface for broadcasting a specialized UI element—a large, screen-centered title—to all players currently connected to the server.

This command acts as a bridge between player or console input and the server's presentation layer. It encapsulates the logic for parsing a complex set of arguments, including flags and optional values, and then uses this data to trigger a global visual event.

A key design choice is its direct dependency on the global Universe object to fetch the complete list of players. This firmly establishes it as a command with server-wide scope and impact. It delegates the low-level task of packet construction and network dispatch to the EventTitleUtil, adhering to the Single Responsibility Principle by separating command parsing from the mechanics of client-server communication.

The argument parsing logic is notably flexible, supporting both formally defined arguments (e.g., --major, --secondary) and a "catch-all" mechanism for the primary title. This allows for simple usage (e.g., `eventtitle My Message`) while still providing advanced control for scripted events.

## Lifecycle & Ownership
-   **Creation:** A single instance of EventTitleCommand is created by the CommandSystem during the server's bootstrap phase. It is registered against the command name "eventtitle" and persists for the server's entire runtime.
-   **Scope:** The command object itself is a long-lived, stateless handler. Its scope is effectively that of a singleton, managed by the command registry. It is not instantiated per-execution.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandSystem clears its registry.

## Internal State & Concurrency
-   **State:** The EventTitleCommand instance is **immutable**. Its fields are final objects that define the command's argument structure (e.g., primaryTitleArg, majorFlag). All state related to a specific execution is passed in via the CommandContext object and is confined to the stack of the executeSync method. The command does not cache data or maintain state between invocations.

-   **Thread Safety:** **This class is not thread-safe and must not be treated as such.** The executeSync method is explicitly designed to run on the main server thread. It directly accesses and iterates over the global player list from `Universe.get()`, which is not a thread-safe collection.

    **WARNING:** Invoking executeSync from any thread other than the main server thread will result in critical race conditions, likely causing a ConcurrentModificationException and potentially corrupting server state. The CommandSystem framework guarantees that all calls are correctly synchronized to the main game loop.

## API Surface
The public contract is almost entirely defined by its base class. Direct interaction with instances of this class is discouraged; interaction should occur via the CommandSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(P) | Executes the command logic. Parses arguments from the context and broadcasts the title to all *P* players on the server. Throws exceptions on invalid context. |

## Integration Patterns

### Standard Usage
This command is not intended to be instantiated or called directly. It is invoked by the server's CommandSystem in response to player chat input or console commands. Programmatic invocation should always be routed through the dispatcher.

```java
// Example of a system (e.g., a quest manager) dispatching the command
CommandSystem commandSystem = server.getService(CommandSystem.class);
CommandSource console = server.getConsoleSource();

// The command string is dispatched for parsing and execution
commandSystem.dispatch(console, "eventtitle \"Zone Discovered\" --secondary=\"The Whispering Valley\" --major");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EventTitleCommand()`. The command framework is responsible for the object's lifecycle. Manually creating an instance will result in an un-registered command that does nothing.
-   **Direct Invocation:** Avoid calling `command.executeSync(context)` directly. This bypasses the permission system, logging, and thread safety guarantees provided by the `CommandSystem.dispatch` method.
-   **Stateful Decoration:** Do not attempt to wrap this class or store state related to its execution. The command is designed to be a stateless handler.

## Data Pipeline
The flow of data for a typical invocation begins with user input and terminates with a network packet being sent to every connected client.

> Flow:
> Player Chat Input -> Server Network Layer -> CommandSystem Parser -> **EventTitleCommand.executeSync** -> Universe (Fetch All Players) -> EventTitleUtil -> Network Packet Encoder -> All Client Network Connections -> Client UI Render


---
description: Architectural reference for SayCommand
---

# SayCommand

**Package:** com.hypixel.hytale.server.core.console.command
**Type:** Transient

## Definition
```java
// Signature
public class SayCommand extends CommandBase {
```

## Architecture & Concepts
The SayCommand class is a concrete implementation of the Command Pattern, designed to handle server-wide broadcast messages. It serves as a direct interface for administrators and systems to communicate with all connected players and the server console simultaneously.

Architecturally, it resides within the server's command processing system. Its primary responsibility is to parse a raw input string, construct a formatted Message object, and distribute it. The class contains logic to differentiate between two input formats:
1.  **Plain Text:** Standard string arguments are wrapped in a predefined chat format, attributing the message to the sender.
2.  **JSON Format:** A raw string beginning with a curly brace is parsed as a structured JSON message, allowing for advanced formatting like colors, translations, and interactive components.

This command directly interfaces with the **Universe** service, the top-level container for all game worlds, to ensure complete message propagation.

## Lifecycle & Ownership
-   **Creation:** An instance of SayCommand is created by the server's central command registration system during the bootstrap sequence. Commands are typically discovered and instantiated once at startup.
-   **Scope:** The object is a stateless singleton for the command definition. It persists for the entire lifetime of the server process.
-   **Destruction:** The instance is garbage collected upon final server shutdown. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** SayCommand is effectively stateless. The only internal field, SAY_COMMAND_COLOR, is a static final constant. All operational data is provided via the CommandContext argument during method invocation.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the core logic is executed within the **executeSync** method, which signals that it must be called from the main server thread to safely interact with game state entities like the Universe and Players. Unsynchronized access from other threads will lead to severe concurrency violations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SayCommand() | constructor | O(1) | Initializes the command definition with its name, aliases, and description key. |
| executeSync(context) | void | O(N) | Executes the broadcast logic. Complexity is linear, proportional to the total number of players (N) in the Universe. |

## Integration Patterns

### Standard Usage
A developer or system should not invoke this class directly. Instead, they should dispatch the command through the server's central command handler. The handler is responsible for parsing the input, locating the correct registered command, checking permissions, and invoking it with the appropriate context.

```java
// Correct invocation is performed by the command system
// Example: A system wants to broadcast a welcome message.

CommandSystem commandSystem = server.getCommandSystem();
CommandSender console = ConsoleSender.INSTANCE;

// The system dispatches the command string, it does not call the class directly
commandSystem.dispatch(console, "say Welcome to the server!");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use *new SayCommand()* in application logic. The command registration system manages the lifecycle of command objects. Creating a new instance will result in an object that is not registered and therefore cannot be invoked by users.
-   **Direct Invocation:** Avoid calling the *executeSync* method directly. Bypassing the server's command dispatcher can lead to permission bypasses, incorrect sender attribution, and critical thread safety violations if called from an asynchronous task.

## Data Pipeline
The flow of data for a typical *say* command execution is unidirectional, originating from a user input and terminating at all connected clients.

> Flow:
> Console Input or Network Packet -> Command Dispatcher -> **SayCommand.executeSync** -> Message Object Creation -> Universe Query -> Player & Console Message Send -> Network Serialization -> Client UI Update


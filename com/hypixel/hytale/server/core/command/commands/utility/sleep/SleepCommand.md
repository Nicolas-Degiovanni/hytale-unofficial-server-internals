---
description: Architectural reference for SleepCommand
---

# SleepCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.sleep
**Type:** Singleton

## Definition
```java
// Signature
public class SleepCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The SleepCommand class serves as a structural component within the server's command processing system. It is an implementation of the Composite pattern, acting as a non-terminal node in the command tree. Its sole responsibility is to group related sub-commands—specifically SleepOffsetCommand and SleepTestCommand—under a single, user-facing namespace: *sleep*.

This class does not contain any execution logic itself. When a player or system issues a command like `/sleep test`, the central CommandManager first routes the request to this SleepCommand instance based on the "sleep" token. SleepCommand, by virtue of its parent AbstractCommandCollection, then delegates the remainder of the command processing to the appropriate registered sub-command, in this case, SleepTestCommand.

This architectural pattern is critical for creating a clean, hierarchical, and extensible command-line interface for server administrators and developers. It prevents namespace pollution in the global command registry and provides a clear organizational structure.

### Lifecycle & Ownership
-   **Creation:** A single instance of SleepCommand is instantiated by the server's central CommandRegistry during the server bootstrap phase. This is typically part of an automated discovery process that scans for all classes implementing a specific command interface or annotation.
-   **Scope:** The instance is a long-lived singleton. It persists for the entire operational lifetime of the server.
-   **Destruction:** The object is eligible for garbage collection only upon server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** The SleepCommand object is effectively **immutable** after its constructor completes. Its internal state consists of a list of its sub-commands, which is populated once and never modified thereafter. It holds no session-specific or player-specific data.
-   **Thread Safety:** This class is inherently **thread-safe**. Because its internal state is fixed upon construction, multiple threads (e.g., different player command-handler threads) can safely traverse this object simultaneously to resolve sub-commands without locks or synchronization primitives.

## API Surface
The public contract is minimal and primarily driven by its constructor. All command execution and sub-command lookup logic is handled by the inherited implementation from AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SleepCommand() | Constructor | O(1) | Instantiates the command collection. Sets the primary command name ("sleep") and description key. Registers all child commands. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The system's CommandRegistry is responsible for its creation and registration. The class is used declaratively.

```java
// Example of how the CommandRegistry might register this command
// This code is conceptual and resides within the command system itself.

CommandRegistry registry = server.getCommandRegistry();
registry.register(new SleepCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SleepCommand()` in game logic or other services. This creates an orphan object that is not registered with the command system and cannot be executed. All commands must be managed exclusively by the CommandRegistry.
-   **State Modification:** Do not attempt to use reflection or other means to add or remove sub-commands after construction. The command hierarchy is intended to be static for the server's lifetime.

## Data Pipeline
The SleepCommand acts as a routing point in the data flow of command execution. It does not transform data but directs it based on command tokens.

> Flow:
> Player Input (`/sleep test`) -> Network Layer -> Server Command Parser -> **SleepCommand** (resolves "test" token) -> SleepTestCommand (receives execution context) -> Game World State Change


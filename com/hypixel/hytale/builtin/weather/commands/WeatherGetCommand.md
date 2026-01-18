---
description: Architectural reference for WeatherGetCommand
---

# WeatherGetCommand

**Package:** com.hypixel.hytale.builtin.weather.commands
**Type:** Transient

## Definition
```java
// Signature
public class WeatherGetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WeatherGetCommand is a server-side component that implements the Command Pattern. It serves as a specific, read-only endpoint within the server's command processing system, responsible for handling the `/weather get` subcommand.

Its primary function is to query the current weather state of a specific World. It does not modify state but instead acts as an inspector, retrieving information from the world's associated WeatherResource. This class forms a critical link between user-facing server administration commands and the internal, data-driven weather simulation system. By extending AbstractWorldCommand, it inherits the necessary boilerplate for world and entity store resolution, allowing it to focus solely on the logic of retrieving and formatting weather information.

## Lifecycle & Ownership
- **Creation:** A single instance of WeatherGetCommand is instantiated by the server's command registration system during the server bootstrap phase. It is registered as a subcommand under a parent `weather` command.
- **Scope:** The command object itself is a singleton that persists for the entire server session. Its `execute` method is invoked on a per-call basis, with a new context provided for each invocation.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. Its only instance field is a `static final` Message object, which is immutable. All operational data, such as the target world and command context, is passed as method parameters to the `execute` method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. The `execute` method is designed to be called by the server's main tick loop for the corresponding world. Direct, multi-threaded invocation of `execute` on the same World instance is not supported and would violate the engine's threading model. The underlying WeatherResource is assumed to be accessed only from its world's dedicated thread.

## API Surface
The public API is defined by its role as a command and is not intended for direct programmatic use outside the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(1) | The command's entry point. Retrieves the WeatherResource from the provided store, reads the forced weather state, and sends a formatted message to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is automatically executed by the server's command dispatcher when a user with appropriate permissions issues the corresponding command.

```java
// CONCEPTUAL: How the command system invokes this class
// This code does not appear in the game logic; it is handled by the engine.

CommandDispatcher dispatcher = server.getCommandDispatcher();
CommandSource source = getPlayerCommandSource("somePlayer");

// User types: /weather get
dispatcher.execute("weather get", source);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WeatherGetCommand()`. The command system manages the lifecycle of command objects. Creating your own instance serves no purpose as it will not be registered or executable.
- **Manual Execution:** Never call the `execute` method directly. Bypassing the command dispatcher will skip critical permission checks, argument parsing, and context setup, leading to NullPointerExceptions and inconsistent world state.

## Data Pipeline
The flow of data for this command is a simple, linear read operation. It translates a user request into a query against the world state and formats the result as a chat message.

> Flow:
> Player Chat Input (`/weather get`) -> Network Layer -> Command Parser -> **WeatherGetCommand.execute()** -> World EntityStore lookup -> Read WeatherResource state -> Message formatting -> CommandContext.sendMessage() -> Network Layer -> Player Chat Output


---
description: Architectural reference for WeatherCommand
---

# WeatherCommand

**Package:** com.hypixel.hytale.builtin.weather.commands
**Type:** Service / Singleton

## Definition
```java
// Signature
public class WeatherCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WeatherCommand class serves as a command aggregator within the server's command processing system. It does not implement any game logic itself. Instead, its sole responsibility is to act as a routing namespace for all weather-related sub-commands, such as setting, querying, or resetting the in-game weather.

This class is an implementation of the Composite design pattern. It groups a collection of leaf command objects (WeatherSetCommand, WeatherGetCommand) under a single, unified interface presented to the command dispatcher. When a player executes a command beginning with *weather*, the central CommandSystem delegates the request to this object, which in turn parses the subsequent arguments to identify and invoke the appropriate sub-command.

This architectural choice decouples the root command from its child operations, allowing for modular and extensible command structures. New weather-related functionality can be added by creating a new sub-command class and registering it within the WeatherCommand constructor, without altering the core command dispatch system.

## Lifecycle & Ownership
- **Creation:** A single instance of WeatherCommand is instantiated by the server's core CommandSystem during the server bootstrap phase. This is typically part of an automated discovery and registration process that scans the classpath for command definitions.
- **Scope:** The instance is a long-lived singleton that persists for the entire server session. It is stored within the central command registry and remains active until server shutdown.
- **Destruction:** The object is eligible for garbage collection only when the server is shutting down and the CommandSystem clears its command registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after construction. The list of sub-commands is populated once within the constructor and is not modified during the object's lifetime. It holds no session-specific or world-specific data.
- **Thread Safety:** The class is inherently thread-safe due to its immutable nature. Command execution is managed by the parent CommandSystem, which is responsible for ensuring that command logic is executed on the appropriate server thread, typically the main game tick thread. Calls into this object from any thread are safe, as they only involve reading the pre-configured command structure.

## API Surface
The public contract is almost entirely defined by its parent, AbstractCommandCollection. The constructor is the primary point of interest.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WeatherCommand() | Constructor | O(1) | Instantiates the command group. Registers a fixed set of sub-commands for weather management. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is not intended. The server's CommandSystem automatically discovers and registers it. A user or system interacts with it via the server console or in-game chat.

The command system will parse the user input and delegate to this handler.

*Example User Interaction:*
```plaintext
/weather set clear
/weather get
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WeatherCommand()`. The object will be unmanaged and will not be registered with the server's command dispatcher, rendering it non-functional.
- **State Modification:** Do not attempt to modify the internal sub-command list after construction via reflection or other means. The command system relies on this list being immutable post-initialization.

## Data Pipeline
This class acts as a dispatcher node in the server's command processing pipeline. It receives a partially parsed command and routes it to the correct handler.

> Flow:
> User Input (`/weather set clear`) -> Server Network Listener -> CommandSystem Parser -> **WeatherCommand** (Identifies "set") -> WeatherSetCommand (Executes logic) -> World State Change


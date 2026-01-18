---
description: Architectural reference for WeatherSetCommand
---

# WeatherSetCommand

**Package:** com.hypixel.hytale.builtin.weather.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class WeatherSetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WeatherSetCommand class is a server-side component responsible for handling the in-game `/weather set` command. It serves as a direct interface between a user's command input and the world's weather simulation state.

As a subclass of AbstractWorldCommand, it operates within the established command system framework. This framework guarantees that the command is executed with a valid World context and provides access to the world's entity storage. Its primary architectural function is to parse a specific weather asset provided by the user and apply it as a persistent override to the target world's weather system.

This class decouples the command input layer from the underlying weather mechanics. It interacts with two key state-holding components:
1.  **WorldConfig:** Persists the forced weather setting, ensuring it survives server restarts.
2.  **WeatherResource:** A live resource that manages the *current* weather state for a world. Modifying this resource causes an immediate change in the game.

The command's argument, `weatherArg`, leverages the engine's typed argument system (ArgTypes.WEATHER_ASSET) to automatically resolve a string input like "hytale:sunny" into a concrete Weather asset object.

### Lifecycle & Ownership
-   **Creation:** An instance of WeatherSetCommand is created by the server's command registration system during the server bootstrap phase. The system scans for command classes and instantiates them to build a central command registry.
-   **Scope:** A single instance of this class persists for the entire server session. It is a stateless handler whose `execute` method is invoked on-demand.
-   **Destruction:** The object is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The WeatherSetCommand instance itself is effectively stateless regarding its execution logic. It holds an immutable `weatherArg` field which defines the command's argument structure. All stateful data, such as the target world and command context, is passed as parameters to the `execute` method.
-   **Thread Safety:** This class is designed to be executed by the server's main thread, which processes game logic and commands for a given world. The `execute` method and the static `setForcedWeather` method are not thread-safe. They perform direct mutations on core world components (WeatherResource, WorldConfig) which are not designed for concurrent access.

**WARNING:** Calling `setForcedWeather` from an asynchronous task or a different thread will lead to race conditions and world state corruption. All interactions with World objects and their associated resources must be synchronized with the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Overridden entry point for the command system. Parses the weather asset from the context, applies it via `setForcedWeather`, and sends a confirmation message to the command sender. |
| setForcedWeather(world, forcedWeather, accessor) | void | O(1) | Static utility method that performs the core logic of updating the world's weather state. It modifies both the live `WeatherResource` and the persistent `WorldConfig`. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is automatically registered and executed by the server's command system. A user triggers its execution by typing the corresponding command in-game.

The conceptual flow initiated by a user is as follows:
```
// User types in chat: /weather set hytale:rain
// The server's command system then invokes the handler:

// (Conceptual - This code is managed by the engine)
CommandContext context = ...; // Context for the user who ran the command
World world = ...; // The world the user is in
Store<EntityStore> store = ...; // The world's entity store

// The command registry finds the WeatherSetCommand instance and calls execute
weatherSetCommand.execute(context, world, store);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WeatherSetCommand()`. The command system manages the lifecycle of command objects. A manually created instance will not be registered and cannot be executed.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure like permission checks, context setup, and argument validation provided by the command system.
-   **Misusing setForcedWeather:** While `setForcedWeather` is a public static method, it is a low-level utility. Using it directly from other systems can break the principle of single responsibility. If another system needs to change the weather, it should ideally go through a dedicated weather management service, not a command's helper method.

## Data Pipeline
The flow of data for this command begins with user input and results in a change to persistent and in-memory world state.

> Flow:
> Player Command Input (`/weather set ...`) -> Server Command Parser -> Argument Resolver (for `ArgTypes.WEATHER_ASSET`) -> **WeatherSetCommand.execute()** -> `setForcedWeather()` -> Update `WeatherResource` (In-Memory) & `WorldConfig` (Persistent) -> World State Change


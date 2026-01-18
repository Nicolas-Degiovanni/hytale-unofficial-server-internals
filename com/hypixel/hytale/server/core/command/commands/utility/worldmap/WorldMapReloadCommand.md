---
description: Architectural reference for WorldMapReloadCommand
---

# WorldMapReloadCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapReloadCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldMapReloadCommand is a concrete implementation within the server's command processing system. It serves as a high-level administrative tool, providing an explicit entry point for operators to trigger a "hot-reload" of a world's map generation logic.

Architecturally, this class acts as a bridge between the user-facing command interface (console or in-game chat) and the core world simulation state. Its primary responsibility is to interact with the active World instance, access its configuration and state managers, and orchestrate the re-initialization of the IWorldMap generator. This allows for dynamic updates to map generation without requiring a full server restart, which is critical for live server maintenance and development workflows.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldMapReloadCommand is instantiated by the server's command registration system during the initial bootstrap sequence. It is registered under the "reload" subcommand within the "worldmap" command group.
- **Scope:** The command object itself is a long-lived singleton that persists for the entire server session. However, its execution context is transient; the execute method is invoked for a brief period and does not persist state between calls.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. All necessary state, such as the target World and the execution CommandContext, is provided as arguments to the execute method. The static MESSAGE field is an immutable constant.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the operations performed within the execute method are **not** safe for concurrent execution. It directly mutates the state of the World and WorldMapManager objects, which are critical, non-thread-safe components of the main game loop. The server's command system is responsible for ensuring that all command executions are serialized and occur on the main server thread to prevent race conditions and data corruption.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | I/O Bound | Triggers the world map reload sequence. Retrieves the map provider from the world configuration, creates a new generator, and installs it into the WorldMapManager. Throws and logs a WorldMapLoadException on failure. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is designed to be invoked by a server administrator through the server console or an in-game entity with sufficient permissions.

```java
// Example invocation via server console or chat
/worldmap reload
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapReloadCommand()`. The command framework handles the lifecycle of command objects. Manually creating an instance will result in a non-functional object that is not registered with the dispatcher.
- **Manual Invocation:** Calling the `execute` method directly from other server systems is a severe design violation. This bypasses the command system's critical cross-cutting concerns, such as permission checking, argument parsing, and context setup. All command logic must flow through the central command dispatcher.

## Data Pipeline
The data flow for this command is initiated by an external actor and results in a state change within the core server simulation.

> Flow:
> Command String -> Command Parser -> **WorldMapReloadCommand.execute()** -> World.getWorldConfig() -> IWorldMapProvider.getGenerator() -> World.getWorldMapManager().setGenerator() -> CommandContext.sendMessage() -> Client/Console Output


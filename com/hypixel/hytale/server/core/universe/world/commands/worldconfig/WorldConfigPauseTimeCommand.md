---
description: Architectural reference for WorldConfigPauseTimeCommand
---

# WorldConfigPauseTimeCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient

## Definition
```java
// Signature
public class WorldConfigPauseTimeCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldConfigPauseTimeCommand is a concrete implementation of the Command Pattern, designed to integrate with the server's command processing system. It encapsulates a single, specific server action: toggling the paused state of a world's game time.

As a subclass of AbstractWorldCommand, it plugs into a framework that handles command registration, permission checking, and argument parsing. Its primary architectural role is to serve as a bridge between a user-initiated action (a player typing a command) and the internal state of a specific World instance.

The class decouples the command invocation logic from the state modification logic. The `execute` method handles the context provided by the command system, while the static `pauseTime` method contains the core business logic. This separation allows other server systems to programmatically pause time using the static method without needing to simulate a full command execution context.

## Lifecycle & Ownership
- **Creation:** A single instance of this class is typically created by the server's command registration system during the server bootstrap phase. It is not instantiated on a per-request basis.
- **Scope:** The command object persists for the entire lifecycle of the server. It is registered once and remains available until the server shuts down.
- **Destruction:** The object is eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields and all required data (World, CommandSender, etc.) is passed as arguments to its methods. This design ensures that a single instance can safely handle concurrent command executions, should the system ever support it.

- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the operations it performs are not. It directly mutates the state of the WorldConfig object.

    **WARNING:** Callers must ensure that modifications to WorldConfig and access to WorldTimeResource are synchronized or executed exclusively on the main game thread. The call to `worldConfig.markChanged()` implies that other systems may react to this state change, creating a high potential for race conditions if this command is invoked from an asynchronous context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Framework entry point. Extracts context and delegates to the static `pauseTime` method. |
| pauseTime(sender, world, store) | void | O(1) | Toggles the game time pause state on the target WorldConfig and sends a confirmation message. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by developers. It is automatically discovered and used by the server's command handling system. The only supported programmatic interaction is via its static method.

```java
// To programmatically pause time for a world
// Note: Ensure this is called from the main server thread.
World targetWorld = ...;
Store<EntityStore> entityStore = ...;
CommandSender console = server.getConsoleSender();

WorldConfigPauseTimeCommand.pauseTime(console, targetWorld, entityStore);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldConfigPauseTimeCommand()`. The instance is managed by the command system. Creating your own instance has no effect as it will not be registered to handle any command strings.
- **Directly Calling Execute:** Do not call the `execute` method directly. This bypasses the command system's processing and permission pipeline. If you need to invoke the logic, use the public static `pauseTime` method.

## Data Pipeline
This component acts as a state mutator triggered by external input. It does not process a continuous stream of data but rather acts on a discrete event.

> **Command Execution Flow:**
> Player Input (`/pausetime`) → Network Packet → Server Command Parser → **WorldConfigPauseTimeCommand.execute()** → **WorldConfigPauseTimeCommand.pauseTime()** → WorldConfig.setGameTimePaused(newState)

> **Feedback Flow:**
> **WorldConfigPauseTimeCommand.pauseTime()** → CommandSender.sendMessage(message) → Network Packet → Client Chat UI Update


---
description: Architectural reference for WorldMapViewRadiusGetCommand
---

# WorldMapViewRadiusGetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapViewRadiusGetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The WorldMapViewRadiusGetCommand is a concrete implementation within the server's Command System framework. It represents a specific, read-only server command that allows an authorized user to query the world map view radius for a target player.

Architecturally, this class adheres to the **Command Pattern**. It encapsulates a request (querying the view radius) as an object, decoupling the command issuer from the object that performs the action. Its inheritance from AbstractTargetPlayerCommand is critical; the base class abstracts away the complex and error-prone logic of parsing and resolving a target player from command arguments. This allows WorldMapViewRadiusGetCommand to focus solely on its core business logic: interacting with the Player and WorldMapTracker components.

This command acts as a data bridge, translating a user-initiated text command into a direct query against the server's entity-component state and formatting the result into a user-friendly message.

## Lifecycle & Ownership
- **Creation:** A single instance of this class is instantiated by the Command System's registry during server bootstrap. The system scans for command definitions and registers them for later dispatch.
- **Scope:** The object is a de-facto singleton managed by the Command System. The same instance is reused for every execution of the corresponding command for the entire duration of the server's runtime.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the Command System registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All data required for its operation, such as the target player and the command context, is provided as arguments to the `execute` method. This design makes the command object itself inherently reusable and predictable.

- **Thread Safety:** The class is thread-safe. As a stateless object, concurrent invocations of the `execute` method will not interfere with each other. However, the responsibility for ensuring that the underlying components (Player, WorldMapTracker, World) are accessed in a thread-safe manner lies with the calling Command System. Typically, all command executions are marshaled onto the main server thread or a specific world's thread to prevent race conditions when accessing game state.

## API Surface
The primary interaction surface is the `execute` method, which is the contract fulfilled for the AbstractTargetPlayerCommand framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldMapViewRadiusGetCommand() | constructor | O(1) | Initializes the command with its name ("get") and description key. |
| execute(...) | protected void | O(1) | Framework-invoked method. Retrieves the WorldMapTracker for the target player, reads the effective and override view radius, and sends a formatted message to the command source. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly by other systems. It is invoked exclusively by the server's command dispatcher in response to a player or console input.

```
// This is a conceptual example of how the framework uses the command.
// A developer would NOT write this code.

// User types: /worldmap viewradius get SomePlayer
CommandDispatcher.dispatch("worldmap viewradius get SomePlayer", commandSource);

// The dispatcher would then resolve this to the WorldMapViewRadiusGetCommand
// instance and invoke its execute method with the correct context.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapViewRadiusGetCommand()` in your code. The command system manages a single, shared instance. Creating new ones is wasteful and bypasses the registration system.
- **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses critical framework functionality, including permission checks, argument parsing, and context setup. To trigger a command programmatically, use the server's primary command dispatching service.

## Data Pipeline
This command facilitates a simple query-response data flow, originating from a user and terminating with a message sent back to that user.

> Flow:
> User Command Input -> Network Layer -> Command Dispatcher -> **WorldMapViewRadiusGetCommand** -> EntityStore Query -> WorldMapTracker State Read -> Message Formatter -> CommandContext -> Network Layer -> Client Chat UI


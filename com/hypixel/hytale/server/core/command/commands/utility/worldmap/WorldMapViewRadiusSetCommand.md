---
description: Architectural reference for WorldMapViewRadiusSetCommand
---

# WorldMapViewRadiusSetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapViewRadiusSetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The WorldMapViewRadiusSetCommand is a server-side command handler responsible for a single, specific administrative action: modifying a player's world map view radius. It operates within the server's Command System, which parses text input from players or the console and routes it to the appropriate handler.

This class acts as the terminal node in the command processing chain for `/worldmap viewradius set`. Its primary function is to serve as a controlled interface between an authorized user and the internal game state of a target player entity. It encapsulates the necessary business logic, including:

*   **Argument Parsing:** Defining and extracting the required `radius` and optional `bypass` flag from the command input.
*   **Permission Enforcement:** The `bypass` flag is explicitly tied to a server permission node, ensuring that only privileged users can set excessively large view radii.
*   **Input Validation:** Applying engine-defined constraints, such as preventing negative values or enforcing a maximum radius for non-bypassing users.
*   **State Mutation:** Directly modifying the `WorldMapTracker` component associated with the target `Player` entity.
*   **User Feedback:** Reporting success or failure back to the command issuer through the provided `CommandContext`.

By extending AbstractTargetPlayerCommand, it inherits the foundational logic for resolving a player target from the command's arguments, simplifying its own implementation to focus solely on the core action.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldMapViewRadiusSetCommand is instantiated by the Command Registry during server bootstrap. The registry scans for command classes and constructs them, building a map of command strings to handler instances.
- **Scope:** The object instance is a long-lived singleton managed by the Command Registry. It persists for the entire server session. However, its operational context is transient; the `execute` method is invoked for a brief moment to handle a single command and does not persist state between calls.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down or the command is explicitly unregistered from the Command Registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after construction. Its fields, `radiusArg` and `bypassArg`, are configuration objects that define the command's signature. They are initialized in the constructor and never change. The class holds no state related to any specific execution.
- **Thread Safety:** **Not thread-safe.** Command execution is expected to occur exclusively on the main server thread. The `execute` method directly accesses and mutates components of a live game entity (`Player`). Invoking this method from any other thread would bypass the server's tick synchronization, leading to critical race conditions, data corruption, and server instability. All interactions with this class must be managed by the server's CommandManager to ensure thread safety.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system. Direct calls are strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, source, ref, player, world, store) | void | O(1) | Validates and applies the view radius change to the target player. Throws exceptions on critical failures; sends user-facing error messages for invalid input. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is automatically registered and invoked by the server's command processing system. A user with appropriate permissions triggers its execution by typing the command into the game client's chat console.

```
# Console or in-game command executed by a player
/worldmap viewradius set PlayerName 128

# Command to set a radius larger than the default maximum, requiring
# the 'server.commands.worldmap.viewradius.set.bypass' permission.
/worldmap viewradius set PlayerName 1024 --bypass
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapViewRadiusSetCommand()`. The resulting object will not be registered with the command system and will be completely non-functional. The server manages the lifecycle of all command handlers.
- **Manual Execution:** Never call the `execute` method directly. Doing so bypasses the entire command pipeline, including critical steps like permission validation, argument parsing from raw text, and target entity resolution. This will result in `NullPointerException` or other unpredictable runtime failures.

## Data Pipeline
The flow of data for a successful command execution begins with user input and terminates with a change in game state and user feedback.

> Flow:
> Player Chat Input -> Server Network Packet -> CommandManager (Parsing & Routing) -> **WorldMapViewRadiusSetCommand.execute()** -> Player.WorldMapTracker (State Mutation) -> CommandContext.sendMessage() -> Server Network Packet -> Player Chat Feedback


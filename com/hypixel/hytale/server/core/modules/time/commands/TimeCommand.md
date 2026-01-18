---
description: Architectural reference for TimeCommand
---

# TimeCommand

**Package:** com.hypixel.hytale.server.core.modules.time.commands
**Type:** Transient

## Definition
```java
// Signature
public class TimeCommand extends AbstractWorldCommand {
```

## Architecture & Concepts

The TimeCommand class is the authoritative server-side implementation for the user-facing `/time` command. It serves as a high-level entry point for players and server administrators to query and manipulate the in-game time of a specific world.

Architecturally, this class follows the **Command Pattern**. It encapsulates a request (e.g., "set time to noon") as an object, thereby decoupling the requester of an action from the object that performs the action. It does not contain any time-keeping logic itself. Instead, it acts as a controller that validates input, checks permissions, and delegates the core logic to the appropriate world-level system, primarily the **WorldTimeResource**.

The class employs a **Composite Pattern** for its subcommands. The main `TimeCommand` acts as a root node, and its constructor registers multiple inner classes, each representing a specific subcommand (`set`, `pause`, `dilation`). This creates a hierarchical command structure that is parsed and executed by the server's central command system. This design makes the command extensible, as new time-related functions can be added by simply creating a new subcommand class and registering it in the constructor.

## Lifecycle & Ownership

-   **Creation:** A single instance of TimeCommand is instantiated by the server's CommandSystem during the server bootstrap phase or when its parent module is loaded. The system scans for classes extending command base types and registers them for use.
-   **Scope:** The registered TimeCommand object is a long-lived singleton that persists for the entire server session. However, its `execute` method is invoked on a per-call basis, operating on a transient CommandContext specific to each invocation.
-   **Destruction:** The object is dereferenced and garbage collected when the server shuts down or the command is explicitly unregistered from the CommandSystem.

## Internal State & Concurrency

-   **State:** The TimeCommand class is effectively **stateless and immutable** post-construction. Its internal fields consist of final references to its subcommand instances and argument definitions, which are configured once in the constructor and are not modified during its lifecycle. All stateful operations are performed on external objects, namely the World and its associated WorldTimeResource.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on a world's main update thread. The server's command processing pipeline guarantees this execution context. Any attempt to invoke its `execute` method from an external or asynchronous thread will lead to race conditions and severe world state corruption.

## API Surface

The primary contract is the `execute` method, which is invoked by the command system. The constructor defines the command's structure and is part of its declarative API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the base command. Retrieves and sends the current world time information to the command sender. |
| SetTimeHourCommand.execute(...) | void | O(1) | Sets the world's time to a specific hour provided as a command argument. Modifies the WorldTimeResource. |
| SetTimePeriodCommand.execute(...) | void | O(1) | Sets the world's time to a predefined period (Dawn, Midday, etc.). Modifies the WorldTimeResource. |
| TimeDilationCommand.execute(...) | void | O(1) | Sets the world's time dilation factor, controlling the speed of the day-night cycle. |
| TimePauseCommand.execute(...) | void | O(1) | Toggles the paused state of the world's game clock. Delegates to WorldConfigPauseTimeCommand. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with this class directly via code. Its primary integration is declarative: the class is discovered and registered by the server's command system. A user then triggers its execution through the game client or server console.

The following example illustrates a hypothetical invocation by the command system after parsing user input.

```java
// PSEUDOCODE: How the CommandSystem invokes this command
CommandContext context = createFromUserInput("/time set 12.5");
World targetWorld = context.getWorld();
Store<EntityStore> store = targetWorld.getStore();

// The system finds the registered TimeCommand instance and its subcommand
TimeCommand.SetTimeHourCommand subCommand = findSubCommand(timeCommand, "set");

// Execution is dispatched
subCommand.execute(context, targetWorld, store);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new TimeCommand()` in game logic. Command objects are managed by the central command registry. Instantiating one manually will have no effect as it will not be registered to handle user input.
-   **Asynchronous Execution:** Never call the `execute` method from a separate thread or an asynchronous callback. All interactions with the World and its Store must be synchronized with the world's main update tick.
-   **External State Modification:** Do not attempt to add or remove subcommands after the object has been constructed. The command's structure is intended to be immutable at runtime.

## Data Pipeline

The TimeCommand acts as a processor in the server's command handling pipeline. It translates structured user input into direct state changes within the world's Entity Component System.

> Flow:
> Player Chat Input (`/time pause`) -> Network Packet -> Server Command Parser -> **TimeCommand** -> WorldConfigPauseTimeCommand -> World.WorldConfig -> World State Update -> Network Sync to Clients


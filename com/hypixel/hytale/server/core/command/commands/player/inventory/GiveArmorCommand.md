---
description: Architectural reference for GiveArmorCommand
---

# GiveArmorCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Transient

## Definition
```java
// Signature
public class GiveArmorCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The GiveArmorCommand class is a server-side implementation of the Command Pattern, designed to handle the in-game `/armor` command. It encapsulates all logic for parsing arguments, querying game data, and modifying player state related to granting armor items.

As a subclass of AbstractAsyncCommand, its core responsibility is to perform potentially long-running operations—such as searching the entire asset database or targeting many players—off the main server thread. This is a critical design choice to prevent server lag (tick stalls) when the command is executed.

It serves as an orchestration layer, interacting with several fundamental server systems:
-   **Command System:** Leverages the argument parsing framework (RequiredArg, OptionalArg, FlagArg) to interpret user input.
-   **Universe:** Queries the central Universe service to resolve player names and references.
-   **Asset System:** Scans the registered Item assets to find armor pieces matching a search query.
-   **World & ECS:** Dispatches the final inventory modification logic to the specific World thread where the target player entity resides, ensuring thread-safe state changes.

## Lifecycle & Ownership
-   **Creation:** A single instance of GiveArmorCommand is created by the server's command registration system during the server bootstrap sequence. It is discovered and instantiated alongside all other command classes.
-   **Scope:** The object instance is a long-lived singleton for the duration of the server's runtime. It is registered once and reused for every execution of the `/armor` command.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
-   **State:** The class instance itself is effectively immutable. Its fields are final argument definitions and static constants. All state related to a specific command execution (e.g., the sender, the provided arguments) is encapsulated within the CommandContext object passed to the `executeAsync` method. This design ensures that the singleton command instance can be safely reused across multiple concurrent executions.
-   **Thread Safety:** This class is thread-safe. The core execution logic is partitioned and dispatched to the appropriate World thread via the `runAsync(..., world)` method. This is the mandated pattern for modifying any world-specific state, such as an entity's inventory. By grouping players by their current World and submitting a separate task for each, it guarantees that all entity component modifications are serialized correctly within that World's game loop, preventing race conditions. The use of `CompletableFuture.allOf` correctly orchestrates these disparate asynchronous tasks before sending a final success message.

## API Surface
The public contract is exclusively defined by its inheritance from AbstractAsyncCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(A + P) | Asynchronously executes the command. Complexity is a function of the number of total game Assets (A) to scan and the number of target Players (P). This method orchestrates player lookups, asset queries, and dispatches inventory modifications to the correct world threads. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or invoke this class directly. The server's command dispatcher handles its invocation when a user or console executes the `/armor` command. The system provides the CommandContext, which contains the parsed arguments and sender information.

```java
// System-level invocation (conceptual)
CommandDispatcher dispatcher = server.getCommandDispatcher();
CommandContext context = dispatcher.parse("armor player_name iron --set");

// The dispatcher finds the GiveArmorCommand instance and calls it
GiveArmorCommand command = commandRegistry.get("armor");
command.executeAsync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new GiveArmorCommand()`. The command system manages the lifecycle of command objects. Direct instantiation will result in a non-functional command that is not registered with the server.
-   **Blocking on Future:** Calling `get()` on the CompletableFuture returned by `executeAsync` from a synchronous, performance-critical thread (like the main server tick) will block that thread and cause severe server lag, defeating the purpose of the asynchronous design.
-   **Cross-Thread State Modification:** Modifying a player's inventory directly within the initial `executeAsync` call is a severe violation of the engine's threading model. All world-specific state changes **must** be dispatched to the corresponding World's thread, as this implementation correctly does with `runAsync`.

## Data Pipeline
The flow of data and control for this command is a multi-stage, asynchronous process.

> Flow:
> User Input (`/armor ...`) -> Command Dispatcher -> Argument Parser -> **GiveArmorCommand.executeAsync** -> Universe (Player Lookup) -> Asset System (Item Query) -> Task Dispatcher (Group Players by World) -> World Thread Pool -> Player Inventory Modification -> Network Packet (Inventory Update) -> Client Render


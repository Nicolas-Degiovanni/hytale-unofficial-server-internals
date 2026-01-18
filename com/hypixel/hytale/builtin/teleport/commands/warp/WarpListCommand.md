---
description: Architectural reference for WarpListCommand
---

# WarpListCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Transient

## Definition
```java
// Signature
public class WarpListCommand extends CommandBase {
```

## Architecture & Concepts
The WarpListCommand class is a server-side command handler responsible for implementing the user-facing `/warp list` command. It serves as a bridge between the command system, the player, and the central warp data store managed by the TeleportPlugin.

Architecturally, this class functions as a controller that exhibits two distinct operational modes based on the execution context:

1.  **UI Mode:** When executed by a player entity without a page number argument, the command defers its response to the client-side UI system. It instantiates and opens a WarpListPage, a dedicated graphical interface for browsing and selecting warps. This offloads the presentation logic to the client and provides a rich user experience.

2.  **Text Mode:** When executed by the server console, or by a player who explicitly provides a page number, the command operates in a text-only mode. It fetches the complete list of warps, performs server-side pagination, and formats the output as a series of chat messages sent directly to the command sender.

This dual-mode design allows the command to serve both interactive players and script-based or administrative console users effectively. It relies entirely on the TeleportPlugin as the single source of truth for warp data, ensuring consistency across the teleportation system.

## Lifecycle & Ownership
-   **Creation:** An instance of WarpListCommand is created by the server's command registration system during the initialization phase of its parent, the TeleportPlugin. It is not instantiated on a per-command basis.
-   **Scope:** The object instance persists for the entire lifecycle of the TeleportPlugin. It remains in memory as long as the `/warp list` command is registered and available on the server.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the TeleportPlugin is disabled or the server shuts down, at which point its commands are unregistered from the system.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its only instance field, pageArg, is a final definition of a command argument and does not change after construction. All warp data is fetched on-demand from the TeleportPlugin during each execution of executeSync, preventing stale data.

-   **Thread Safety:** The class is **not thread-safe** and must be handled exclusively by the server's command processing thread. The primary entry point, executeSync, implies a synchronous execution contract on a designated server thread.

    **WARNING:** To maintain thread safety when interacting with world state, the class uses the `world.execute(() -> ...)` pattern. This schedules the UI-opening logic to run on the specific thread assigned to the player's world, preventing race conditions and ensuring safe access to player components and the PageManager. Direct manipulation of player or world state outside of this scheduled task is a critical error.

## API Surface
The public contract is defined by its inheritance from CommandBase, with the core logic contained within the overridden `executeSync` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N log N) | The primary entry point called by the command system. Complexity is dominated by sorting the warp list when operating in Text Mode, where N is the total number of warps. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by developers. It is automatically registered and executed by the server's command processing system. A hypothetical programmatic execution would involve dispatching the command string through the server's command manager.

```java
// A server plugin programmatically dispatching the command for a player
CommandManager commandManager = server.getCommandManager();
PlayerRef player = ...;
commandManager.dispatchCommand(player, "warp list");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new WarpListCommand()`. The command system manages the lifecycle of command objects. Direct instantiation will result in a non-functional object that is not registered to handle any commands.
-   **External Invocation:** Do not call the `executeSync` method directly. It requires a fully-formed CommandContext object that is only correctly constructed by the server's command processor.
-   **Assuming Warps are Loaded:** Do not interact with this command without first ensuring the TeleportPlugin has completed its data loading. The command includes a guard for this, but dependent systems should be aware of the initialization dependency.

## Data Pipeline
The flow of data and control through this command varies significantly based on its operational mode.

> **UI Mode Flow (Player, no page argument):**
> Player Input (`/warp list`) -> Command Parser -> **WarpListCommand.executeSync** -> TeleportPlugin (Get Warps) -> World Thread Scheduler -> PageManager.openCustomPage(WarpListPage) -> Client UI Render

> **Text Mode Flow (Console or Player with page argument):**
> User Input (`/warp list 2`) -> Command Parser -> **WarpListCommand.executeSync** -> TeleportPlugin (Get Warps) -> Sort & Paginate List -> CommandContext.sendMessage -> Sender Chat Output


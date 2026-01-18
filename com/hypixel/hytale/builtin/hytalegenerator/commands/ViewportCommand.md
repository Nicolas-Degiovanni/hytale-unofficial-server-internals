---
description: Architectural reference for ViewportCommand
---

# ViewportCommand

**Package:** com.hypixel.hytale.builtin.hytalegenerator.commands
**Type:** Transient

## Definition
```java
// Signature
public class ViewportCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ViewportCommand serves as the primary user-facing entry point for the world generation preview system. It is a server-side command that translates player input into a concrete, live-reloading preview region, represented by an NViewport instance.

Architecturally, this class acts as a critical bridge between three core engine systems:
1.  **The Command System:** It registers itself as a command ("Viewport") and parses player arguments like a target radius or a delete flag.
2.  **The Builder Tools System:** When a radius is not specified, it queries the BuilderToolsPlugin for the player's active BlockSelection to define the preview boundaries.
3.  **The Asset Management System:** Its most significant function is to create a refresh task for the NViewport and register it as a listener on the AssetManager. This enables a powerful hot-reload workflow for developers; any change to relevant assets will automatically trigger a regeneration of the previewed area.

This command is central to the iterative world generation development loop, allowing creators to see the effects of their changes in-game without a server restart.

## Lifecycle & Ownership
-   **Creation:** An instance of ViewportCommand is created by the server's plugin or command loader during the server bootstrap phase. Its required AssetManager dependency is injected via its constructor at this time.
-   **Scope:** The ViewportCommand object itself is a long-lived service, persisting for the entire server session. However, its internal state, specifically the `activeTask` field, is highly transient. This field holds a reference to the refresh task for the *last successfully created viewport*.
-   **Destruction:** The command object is destroyed upon server shutdown or when its parent plugin is unloaded. The `activeTask` is effectively destroyed and unregistered from the AssetManager whenever a player executes the command again, either to create a new viewport or to explicitly delete the existing one.

**WARNING:** The `activeTask` field represents a singleton state within the command instance. If multiple players use this command on a server, only the most recently created viewport will be linked to the asset hot-reload system. This design is best suited for single-developer test servers.

## Internal State & Concurrency
-   **State:** The class is **Mutable**. It maintains a single nullable field, `activeTask`, which holds the Runnable responsible for refreshing the active NViewport. This state is overwritten with each new successful command execution.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be operated on by a single thread. The `execute` method is invoked by the server's main command processing thread. The `activeTask` itself is scheduled to run on the primary world thread via `world.execute`. Direct, concurrent invocation of the `execute` method would lead to a race condition on the `activeTask` field, resulting in unpredictable behavior and potential memory leaks if a task is overwritten before being unregistered.

## API Surface
The primary interaction surface is through the command system, which invokes the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(N) | Executes the command logic. Creates, deletes, or replaces the active viewport. Complexity is proportional to N, the number of chunks in the viewport, due to the initial `refresh` call. |

## Integration Patterns

### Standard Usage
Interaction with this class is intended exclusively through the in-game chat or server console. A developer or player provides the command and its arguments, which the server's command system routes to this class's `execute` method.

To create a viewport based on the player's current selection:
```
/viewport
```

To create a viewport in a 5-chunk radius around the player:
```
/viewport radius 5
```

To remove the currently active viewport and disable hot-reloading:
```
/viewport --delete
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually create an instance using `new ViewportCommand(assetManager)`. The object will not be registered with the server's command system and will be non-functional. It must be instantiated and managed by the engine's plugin framework.
-   **State Manipulation:** Do not attempt to access or modify the internal `activeTask` field from other systems. This is a private implementation detail used to manage the connection to the AssetManager and is subject to change.
-   **Server-wide Reliance:** Do not design systems that assume a single, persistent viewport for the entire server. The command's state is tied to the last player who invoked it and should be considered ephemeral.

## Data Pipeline
The command initiates two primary data flows: one for creation and one for the subsequent hot-reload cycle.

**Flow 1: Viewport Creation**
> Player Command Input -> Command System Parser -> **ViewportCommand.execute** -> Bounds Calculation (from Player Position or BlockSelection) -> NViewport Instantiation -> AssetManager.registerReloadListener(task) -> Initial `viewport.refresh()`

**Flow 2: Asset Hot-Reload**
> Filesystem Asset Change -> AssetManager reload trigger -> **ViewportCommand.activeTask.run()** -> World Thread Scheduler (`world.execute`) -> `viewport.refresh()` -> World updated in-place


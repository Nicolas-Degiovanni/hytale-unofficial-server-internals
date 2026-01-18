---
description: Architectural reference for PrefabEditSessionManager
---

# PrefabEditSessionManager

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabEditSessionManager {
```

## Architecture & Concepts
The PrefabEditSessionManager is the central authority for the server's prefab editing functionality. It orchestrates the entire lifecycle of a prefab editing session, acting as a stateful service that manages temporary, isolated worlds for players to safely build and modify prefabs.

Its primary architectural role is to serve as a bridge between user-initiated commands, the core server `Universe` which manages all worlds, and the low-level prefab serialization and pasting systems. The manager ensures that editing sessions are isolated, preventing multiple users from editing the same prefab file simultaneously and providing a sandboxed environment for each editor.

This system is fundamentally asynchronous. It leverages `CompletableFuture` to perform resource-intensive operations like world creation and prefab file I/O without blocking the main server thread. State is carefully managed through a series of internal maps and sets to track active sessions, locked files, and the progress of asynchronous loading operations.

## Lifecycle & Ownership
- **Creation:** A single instance of PrefabEditSessionManager is instantiated by its parent `JavaPlugin` during the server's builder tools plugin initialization. Upon creation, it immediately subscribes to global server events, such as `AddPlayerToWorldEvent` and `PlayerReadyEvent`, to hook into the player connection lifecycle.
- **Scope:** The manager is a long-lived singleton that persists for the entire duration of the server's runtime, or until the parent plugin is disabled. Its internal state evolves as players start and end editing sessions.
- **Destruction:** The manager itself is garbage collected when the parent plugin is unloaded. Individual editing sessions and their associated temporary worlds are destroyed when a player exits the editor via the `exitEditSession` method or when a session is cancelled via `cleanupCancelledSession`. The temporary world's files are deleted from disk upon removal, as configured by `worldConfig.setDeleteOnRemove(true)`.

## Internal State & Concurrency
- **State:** The PrefabEditSessionManager is highly stateful and mutable. It maintains several critical collections:
    - `activeEditSessions`: Maps a player's UUID to their active `PrefabEditSession` object.
    - `prefabsBeingEdited`: A `HashSet` of file paths for prefabs that are currently being edited, acting as a global lock to prevent concurrent modification.
    - `inProgressLoading`: A `HashSet` of player UUIDs who are currently in the process of loading a prefab, used to prevent duplicate session creation requests.
    - `cancelledLoading`: A `HashSet` to flag sessions that a user has requested to cancel during a long-running load operation.

- **Thread Safety:** This class is **not** inherently thread-safe and operates in a complex, multi-threaded environment. It manages concurrency through several patterns:
    - **Asynchronous Operations:** Heavy lifting such as file I/O and world generation is offloaded to worker threads using `CompletableFuture`.
    - **World Thread Scheduling:** Interactions with a specific world's data (e.g., pasting blocks, creating entities) are marshalled onto that world's dedicated thread via `world.execute()`. This is the primary mechanism for ensuring safe mutation of world state.
    - **State Flags:** The `inProgressLoading` and `cancelledLoading` sets act as simple, non-blocking locks to manage the state of the asynchronous session creation pipeline.

    **WARNING:** Direct, concurrent access to the internal state collections from multiple threads without proper synchronization is unsafe. The design assumes that state-mutating entry points like `createEditSession` are called from a context that ensures serialized access per player, such as a command handler.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createEditSessionForNewPrefab(...) | CompletableFuture<Void> | O(N) | Asynchronously creates a new, empty prefab editing world. Complexity is high and dependent on world generation time. |
| loadPrefabAndCreateEditSession(...) | CompletableFuture<Void> | O(N) | Asynchronously creates a world and pastes one or more prefabs into it. Complexity is high, dominated by file I/O and block pasting operations. |
| exitEditSession(...) | CompletableFuture<Void> | O(N) | Asynchronously teleports a player out of an edit session and schedules the temporary world for destruction. |
| cleanupCancelledSession(...) | CompletableFuture<Void> | O(N) | Forcibly cleans up resources for a session that was cancelled during creation. |
| isEditingAPrefab(playerUUID) | boolean | O(1) | Checks if a player has an active session. |
| getPrefabEditSession(playerUUID) | PrefabEditSession | O(1) | Retrieves the session object for a given player. |
| cancelLoading(playerUuid) | void | O(1) | Flags an in-progress loading operation for cancellation. |

## Integration Patterns

### Standard Usage
The manager should be treated as a service retrieved from the parent plugin or a central service registry. The primary interaction is to initiate an editing session in response to a player action, such as executing a server command.

```java
// A command handler initiates a prefab loading session
PrefabEditSessionManager sessionManager = plugin.getSessionManager();
PrefabEditorCreationSettings settings = ... // Populate from command arguments

// The call is non-blocking and returns a future
sessionManager.loadPrefabAndCreateEditSession(
    playerRef,
    playerComponent,
    settings,
    componentAccessor,
    (progress) -> {
        // Optionally send progress updates to the player
        playerRef.sendMessage(Message.literal("Loading: " + progress.getCurrentPhase()));
    }
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PrefabEditSessionManager()`. This would create a rogue manager with isolated state, breaking the global prefab locking mechanism and session tracking. Always use the single, plugin-provided instance.
- **Ignoring Futures:** The session creation and exit methods are asynchronous. Calling them without handling the resulting `CompletableFuture` means you have no way to know if the operation succeeded, failed, or was cancelled.
- **State Tampering:** Do not directly modify the internal collections returned by methods like `getActiveEditSessions`. Modifying this state from outside the manager will lead to session corruption and resource leaks.
- **Blocking on Main Thread:** Do not call `.join()` or `.get()` on the returned futures from the main server thread. This will freeze the server until the entire world generation and prefab pasting process is complete.

## Data Pipeline

The data flow for loading an existing prefab into an editor session is a multi-stage, asynchronous pipeline:

> Flow:
> Player Command -> **PrefabEditSessionManager**.loadPrefabAndCreateEditSession
> 1.  Validate request, check for existing sessions or file locks.
> 2.  Asynchronously create a new `World` instance via `Universe.makeWorld()`.
> 3.  Concurrently, asynchronously load prefab file(s) from disk into an in-memory `IPrefabBuffer` representation.
> 4.  Once both the world and prefab buffers are ready, schedule a task on the new world's thread.
> 5.  Inside the world thread, invoke `PrefabUtil.paste` to translate the `IPrefabBuffer` data into block and entity data within the world's chunks.
> 6.  Create and register the `PrefabEditSession` state object.
> 7.  Schedule a task on the player's original world thread to attach a `Teleport` component, initiating the player's transfer.
> 8.  The `onPlayerAddedToWorld` and `onPlayerReady` event listeners fire, performing final setup like giving the player the editor tool.


---
description: Architectural reference for BuilderToolsPlugin
---

# BuilderToolsPlugin

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Singleton

## Definition
```java
// Signature
public class BuilderToolsPlugin extends JavaPlugin implements SelectionProvider, MetricProvider {
```

## Architecture & Concepts
The BuilderToolsPlugin is the central nervous system for all in-game world editing and building functionalities. As a core JavaPlugin, it is loaded once at server startup and persists for the server's entire lifetime. It serves as the authoritative manager for player-specific editing states, commands, scripted brushes, and clipboard operations.

Its primary responsibilities include:
- **State Management:** Maintains a BuilderState instance for each player, encapsulating their current selection, clipboard contents, and undo/redo history. This state is managed in the builderStates concurrent map, keyed by player UUID.
- **Command Registration:** Registers a comprehensive suite of world editing commands, such as CopyCommand, PasteCommand, SetCommand, and UndoCommand, with the server's CommandRegistry.
- **Brush Operation System:** Defines and registers a wide variety of BrushOperation types. These operations are the fundamental building blocks for creating complex, data-driven scripted brushes, allowing for sophisticated terrain and structure generation tools.
- **Interaction Handling:** Registers custom server-side Interaction handlers to process tool usage packets sent from the client, translating player actions into world modifications.
- **Lifecycle Management:** Hooks into player connection and disconnection events (PlayerConnectEvent, PlayerDisconnectEvent) to manage the lifecycle of BuilderState objects, including persistence and cleanup logic.
- **Selection Authority:** Implements the SelectionProvider interface, making it the canonical source for SelectionManager to retrieve player selection data.

The plugin's architecture is designed around a central, thread-safe singleton that dispatches tasks to per-player state objects. All world-modifying operations are carefully queued and executed on the appropriate world thread to prevent concurrency issues.

### Lifecycle & Ownership
- **Creation:** A single instance of BuilderToolsPlugin is created by the server's plugin loader during the server bootstrap sequence. The static instance field is set within the constructor, enforcing the singleton pattern.
- **Scope:** The plugin instance is global and persists for the entire server session. It is never destroyed or re-created during runtime.
- **Destruction:** The shutdown method is invoked by the plugin loader during server shutdown. This method is responsible for canceling scheduled tasks, such as the cleanupTask which garbage collects expired BuilderState objects.

## Internal State & Concurrency
- **State:** The BuilderToolsPlugin is highly stateful. Its primary state is the builderStates map, a ConcurrentHashMap that associates player UUIDs with their corresponding BuilderState objects. Each BuilderState is mutable, containing the player's selection, clipboard, undo/redo queues, and a task queue for pending operations. Configuration values like historyCount and toolExpireTimeNanos are loaded from a config file and stored as instance fields.

- **Thread Safety:** This class is designed to be thread-safe, which is critical for a system handling concurrent player actions.
    - The top-level builderStates map uses ConcurrentHashMap for safe access and modification from multiple threads.
    - The nested BuilderState class employs StampedLock to protect its internal undo/redo queues and its task queue, preventing race conditions during history manipulation or task scheduling.
    - **CRITICAL:** All operations that modify the world are serialized through a task queue within each BuilderState. The addToQueue method schedules a task to be run via CompletableFuture.runAsync on the player's associated World executor. This pattern is essential to ensure all block and entity modifications occur on the main world thread, preventing data corruption. Direct modification of world state from other threads is strictly forbidden.

## API Surface
The public API provides controlled access to the builder tools system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | BuilderToolsPlugin | O(1) | Returns the global singleton instance of the plugin. |
| getState(player, playerRef) | BuilderState | O(1) | Retrieves or creates the state object for a specific player. This is the primary entry point for accessing a player's editing context. |
| addToQueue(player, playerRef, task) | void | O(1) | Safely schedules a world-modifying task for a player. The task is guaranteed to execute on the correct world thread. |
| computeSelectionCopy(ref, player, task, accessor) | void | O(N) | Implements the SelectionProvider interface. Asynchronously computes a copy of the player's selection and passes it to the provided task. |
| onToolArgUpdate(playerRef, player, packet) | void | O(1) | Handles incoming network packets to update metadata on a builder tool ItemStack. |

## Integration Patterns

### Standard Usage
All interactions with the builder tools system should go through the singleton instance. To perform an operation on behalf of a player, first retrieve their BuilderState and then queue the task for execution.

```java
// How a developer should normally use this
BuilderToolsPlugin plugin = BuilderToolsPlugin.get();
Player player = ...;
PlayerRef playerRef = ...;

// Retrieve the player-specific state
BuilderToolsPlugin.BuilderState state = plugin.getState(player, playerRef);

// Queue a thread-safe operation
state.addToQueue((ref, builderState, accessor) -> {
    // This code will execute on the world thread
    builderState.set(BlockPattern.parse("stone"), accessor);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use new BuilderToolsPlugin(). The server manages its lifecycle. Always use the static get() method.
- **Cross-Thread World Modification:** Do not get a BuilderState and call methods like set, paste, or undo directly from a network thread or any thread other than the target world's main thread. This will lead to severe concurrency bugs and server instability. **Always** use addToQueue for any action that modifies world state.
- **Unsafe State Access:** While BuilderState uses locks, directly accessing its internal collections (like undo or tasks) without acquiring the appropriate StampedLock is a severe anti-pattern that breaks thread safety guarantees.

## Data Pipeline
The flow of data for a typical world editing action, such as using a brush tool, follows a clear path from client input to world modification.

> Flow:
> Client Input (Mouse Click) -> BuilderToolOnUseInteraction Packet -> Server Network Thread -> BuilderToolsPacketHandler -> **BuilderState.edit()** -> ToolOperation created -> BlockSelection modified in memory -> BlockSelection.placeNoReturn() -> World Chunks updated -> ActionEntry pushed to BuilderState undo queue -> Client receives chunk updates


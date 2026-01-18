---
description: Architectural reference for MemoriesPlugin
---

# MemoriesPlugin

**Package:** com.hypixel.hytale.builtin.adventure.memories
**Type:** Singleton

## Definition
```java
// Signature
public class MemoriesPlugin extends JavaPlugin {
```

## Architecture & Concepts
The MemoriesPlugin serves as the central authority and persistence layer for the server-wide "Memories" adventure feature. As a `JavaPlugin`, it is a foundational, self-contained module loaded at server startup. Its primary architectural role is to manage the lifecycle of all memory-related systems, components, and data.

This class operates as a multi-faceted service hub:
1.  **Registry:** It registers all necessary ECS components (PlayerMemories), systems (PlayerAddedSystem), commands (MemoriesCommand), and custom interactions (SetMemoriesCapacityInteraction) with the core engine.
2.  **Service Locator:** It provides a static singleton accessor, `get()`, allowing any other part of the server codebase to interact with the global state of recorded memories.
3.  **Persistence Controller:** It is solely responsible for serializing the set of globally discovered memories to and from the filesystem (memories.json). This ensures that player progression in the Memories system persists across server restarts.
4.  **Data Aggregator:** It collects all available Memory definitions from various registered MemoryProvider instances, creating a canonical, server-wide manifest of all discoverable memories.

The core concept revolves around a global, shared set of `RecordedMemories`. Unlike per-player data, memories recorded by *any* player contribute to a single, universal collection. This plugin is the gatekeeper for that collection.

### Lifecycle & Ownership
- **Creation:** The MemoriesPlugin is instantiated once by the server's plugin loader during the server bootstrap sequence. The static `instance` is set within the constructor, guaranteeing its availability before gameplay begins.
- **Scope:** The instance is a server-wide singleton that persists for the entire runtime of the server. Its state represents a global truth for the game world.
- **Destruction:** The `shutdown` method is invoked by the plugin loader when the server is shutting down. This is a critical lifecycle hook used to guarantee the final state of `RecordedMemories` is flushed to disk. Failure to complete this operation can result in data loss.

## Internal State & Concurrency
- **State:** The internal state is highly mutable and central to the feature's operation.
    - **Configuration State:** Fields like `providers` and `allMemories` are populated during the `setup` and `start` phases. They are considered effectively immutable after server initialization is complete.
    - **Dynamic State:** The `recordedMemories` field is the primary dynamic state. It holds the set of all memories unlocked globally. This object is loaded from disk at startup and is continuously updated during gameplay.

- **Thread Safety:** This class is designed to be thread-safe for its core persistence operations.
    - The internal `RecordedMemories` class encapsulates the `Set<Memory>` behind a `ReentrantReadWriteLock`.
    - All public methods that read from or write to the set of recorded memories (e.g., `hasRecordedMemory`, `recordPlayerMemories`) acquire the appropriate lock before access. This prevents race conditions when multiple player actions attempt to update the global memory state simultaneously.
    - **Warning:** While the core state is thread-safe, collections returned by methods like `getRecordedMemories` are copies. They represent a point-in-time snapshot and should not be held for long periods if up-to-date information is required.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | MemoriesPlugin | O(1) | Provides static access to the singleton instance. |
| registerMemoryProvider(provider) | void | O(1) | Registers a new source of memories. Must be called during plugin setup. |
| recordPlayerMemories(playerMemories) | boolean | O(N + I/O) | Atomically adds a player's collected memories to the global set and persists to disk. This is a write-locked, blocking I/O operation. |
| hasRecordedMemory(memory) | boolean | O(1) | Checks if a specific memory exists in the global set. This is a read-locked operation. |
| getRecordedMemories() | Set<Memory> | O(N) | Returns a defensive copy of all globally recorded memories. This is a read-locked operation. |
| clearRecordedMemories() | void | O(I/O) | Wipes the entire global set of recorded memories and persists the empty state to disk. This is a write-locked, blocking I/O operation. |
| getMemoriesLevel(config) | int | O(N) | Calculates the current server-wide "Memories Level" based on the count of recorded memories. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is to retrieve the singleton instance and query or update the global memory state. This is common in systems that award memories to players.

```java
// In a separate system or plugin...
MemoriesPlugin memoriesPlugin = MemoriesPlugin.get();

// A player has just completed an action that should record memories
PlayerMemories component = ... // get the player's memory component
boolean memoriesWereAdded = memoriesPlugin.recordPlayerMemories(component);

if (memoriesWereAdded) {
    // Potentially trigger other global events
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MemoriesPlugin()`. The server's plugin framework is the exclusive owner of its lifecycle. Always use the static `MemoriesPlugin.get()` method.
- **Post-Initialization Registration:** Do not call `registerMemoryProvider` after the server has finished starting up. The internal memory manifest is built during the `onAssetsLoad` event and will not be updated dynamically.
- **Frequent I/O Operations:** Avoid calling `recordPlayerMemories` or `clearRecordedMemories` in rapid succession or within a high-frequency game loop. Each call triggers a synchronous, blocking write to the filesystem which can cause performance degradation. These methods are designed for discrete, significant game events.

## Data Pipeline
The flow of data for recording a new memory is unidirectional, moving from a specific player entity to a global, persistent state on the server's disk.

> Flow:
> Player Action -> `PlayerMemories` Component is populated -> System calls `recordPlayerMemories()` -> **MemoriesPlugin** acquires write lock -> `RecordedMemories` internal Set is updated -> BsonUtil writes state to `memories.json` -> Lock is released


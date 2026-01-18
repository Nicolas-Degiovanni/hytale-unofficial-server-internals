---
description: Architectural reference for WorldPathConfig
---

# WorldPathConfig

**Package:** com.hypixel.hytale.server.core.universe.world.path
**Type:** Stateful Model

## Definition
```java
// Signature
public class WorldPathConfig {
```

## Architecture & Concepts
The WorldPathConfig class is a data model that represents the collection of named WorldPath objects associated with a specific server World. It is not a service, but rather an in-memory representation of the `paths.json` configuration file stored within a world's save directory.

Its primary architectural function is to serve as the authoritative source for pathing data for a given world instance. It decouples systems that need path information (like AI, quest triggers, or player spawning) from the underlying persistence mechanism.

This class is deeply integrated with the Hytale Codec system, which provides a declarative and type-safe framework for serialization and deserialization. Modifications to the path configuration are broadcast via the server's global event bus, allowing other systems to react to changes dynamically without direct coupling.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. The static factory method `load(World world)` is the designated entry point. This method asynchronously reads from the world's save files. A new, empty WorldPathConfig is created in memory if the `paths.json` file is missing or corrupt, ensuring operational continuity.
- **Scope:** The lifecycle of a WorldPathConfig instance is strictly bound to its parent World object. It is loaded when the World is initialized and persists in memory for the entire duration that the World is active on the server.
- **Destruction:** The object is eligible for garbage collection once the parent World is unloaded and all external references are cleared. Its state is persisted to disk via the `save` method, which is typically invoked as part of the World's shutdown or save-cycle sequence.

## Internal State & Concurrency
- **State:** The class is mutable. Its core state is the `paths` field, a map containing all WorldPath objects. This map serves as a live, in-memory cache of the on-disk configuration. Changes made through the public API are immediately reflected in this state.
- **Thread Safety:** This class is designed to be thread-safe. The internal `paths` map is a ConcurrentHashMap, allowing for safe concurrent read and write operations from multiple server threads. Furthermore, the file I/O operations (`load` and `save`) are fully asynchronous, returning a CompletableFuture. This design prevents blocking the main server thread during potentially slow disk operations, which is critical for server performance.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(World world) | CompletableFuture<WorldPathConfig> | O(N) | **Static Factory.** Asynchronously loads configuration from disk. The primary entry point for creating an instance. |
| save(World world) | CompletableFuture<Void> | O(N) | Asynchronously serializes the current state and writes it to the world's `paths.json` file. |
| getPath(String name) | WorldPath | O(1) | Retrieves a single, named WorldPath. Returns null if not found. |
| getPaths() | Map<String, WorldPath> | O(1) | Returns an unmodifiable view of the entire path map. **Warning:** Do not attempt to cast and modify this map. |
| putPath(WorldPath worldPath) | WorldPath | O(1) | Adds or updates a WorldPath. Dispatches a WorldPathChangedEvent to the server event bus. |
| removePath(String path) | WorldPath | O(1) | Removes a WorldPath by its name. |

## Integration Patterns

### Standard Usage
The WorldPathConfig should be loaded during the World's initialization phase. The returned CompletableFuture should be handled asynchronously to avoid blocking the server's main thread.

```java
// Example from within a World's initialization logic
CompletableFuture<WorldPathConfig> loadFuture = WorldPathConfig.load(this);

// Chain subsequent logic to execute after the config is loaded
loadFuture.thenAcceptAsync(config -> {
    this.pathConfig = config;
    HytaleLogger.getLogger().info("World paths loaded successfully.");
    
    // Other systems can now safely access the configuration
    WorldPath spawn = config.getPath("spawn_point");
    if (spawn != null) {
        // ... use the spawn path
    }
}, this.getServer().getExecutor());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldPathConfig()`. This bypasses the critical `load` logic and results in an empty, disconnected configuration object that does not reflect the world's saved state.
- **Blocking on Futures:** Calling `load(world).join()` or `save(world).get()` on the main server thread is a severe performance anti-pattern. It will freeze the server tick loop while waiting for disk I/O, leading to lag and potential crashes. Always use asynchronous callbacks like `thenAccept` or `whenComplete`.
- **State Mismanagement:** Do not hold a reference to the map returned by `getPaths()` and expect it to update. While the view is unmodifiable, retrieving it once and caching it is unsafe if another thread modifies the configuration via `putPath`. Always call `getPath` or `getPaths` to ensure you have the most current data.

## Data Pipeline

The class manages two primary data flows: loading state from disk and saving state back to disk.

> **Load Flow:**
> `paths.json` on Disk -> `RawJsonReader` -> **WorldPathConfig.CODEC** (Deserialization) -> In-Memory `WorldPathConfig` Instance

> **Save Flow:**
> In-Memory `WorldPathConfig` Instance -> **WorldPathConfig.CODEC** (Serialization) -> BsonValue -> `BsonUtil` -> `paths.json` on Disk

> **Runtime Event Flow:**
> API Call to `putPath()` -> Internal `ConcurrentHashMap` Update -> `HytaleServer.getEventBus()` -> `WorldPathChangedEvent` -> Subscribed Systems (e.g., AI Pathfinding, Quest System)


---
description: Architectural reference for BuilderManager
---

# BuilderManager

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Singleton-scoped Service

## Definition
```java
// Signature
public class BuilderManager {
```

## Architecture & Concepts
The BuilderManager is the central authority for the entire NPC asset pipeline. It acts as a registry, loader, and validator for all NPC configurations, which are defined as JSON files on disk. Its primary responsibility is to transform these declarative JSON assets into live, in-memory, and executable builder objects that the server can use to spawn and manage NPCs.

Architecturally, it serves several key functions:

*   **Asset Discovery & Loading:** It scans designated asset directories for NPC JSON files, parsing them into a structured format.
*   **Factory-Driven Instantiation:** It employs a **Factory Pattern** via `BuilderFactory` implementations. When a JSON file is loaded, the manager uses a registered factory corresponding to the asset's `Class` property to create the appropriate `Builder` object (e.g., `Role`, `Behavior`, `Component`). This makes the system extensible to new NPC types without modifying the manager itself.
*   **Indexing & Caching:** For high-performance access, every loaded NPC configuration is assigned a unique integer index. The manager maintains a mapping from the asset's string name to this index, and a concurrent cache mapping the index back to the `BuilderInfo` object. Game systems should always reference NPCs by their integer index to avoid string comparisons at runtime.
*   **Dependency Graph Management:** NPCs can be composed of other components, creating a complex dependency graph. The manager is responsible for resolving and validating this graph, detecting critical errors like cyclic dependencies or references to missing builders.
*   **Live Reloading:** It integrates with the `AssetMonitor` service to detect real-time changes to NPC JSON files on disk. This allows developers to modify NPC configurations while the server is running and see the changes immediately, a critical feature for rapid iteration.

## Lifecycle & Ownership
- **Creation:** A single instance of BuilderManager is created and owned by the `NPCPlugin` during the server's primary bootstrap sequence. It is a foundational service for all NPC-related functionality.
- **Scope:** The instance persists for the entire server session. Its cached data represents the definitive source of truth for all available NPC configurations.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the `NPCPlugin` is unloaded, typically during a server shutdown. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The BuilderManager is highly stateful and mutable. Its core state consists of:
    - `builderCache`: A concurrent map holding all loaded and processed `BuilderInfo` objects, keyed by their integer index.
    - `nameToIndexMap`: A map from an NPC's string name to its integer index.
    - `factoryMap`: A map of registered `BuilderFactory` instances.

- **Thread Safety:** The class is designed to be thread-safe for most read operations but requires careful synchronization for writes.
    - **CRITICAL:** The `nameToIndexMap` is protected by a `ReentrantReadWriteLock`. This allows multiple threads to read the map concurrently (e.g., `getIndex`), but write operations (`getOrCreateIndex`, `cacheBuilder`) acquire an exclusive lock. This is a performance optimization for the common case of reading data.
    - The `builderCache` is an `Int2ObjectConcurrentHashMap`, which provides built-in thread safety for atomic operations like `put` and `get`.
    - **WARNING:** While the underlying collections are thread-safe, complex operations like `loadBuilders` or `loadFile` are not atomic. Calling these methods concurrently from multiple threads will result in unpredictable behavior and data corruption. They are intended to be executed from a single, controlled thread, such as the main server thread or a dedicated asset loading thread.

## API Surface
This section highlights the key public methods that form the contract of the BuilderManager.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerFactory(factory) | void | O(1) | Registers a `BuilderFactory` for a specific NPC category. Must be called at startup. |
| loadBuilders(pack, includeTests) | boolean | O(N * D) | Scans an `AssetPack`, loads all NPC files, and validates them. N is files, D is dependency depth. |
| getOrCreateIndex(name) | int | O(1) | Retrieves the integer index for a named NPC, creating one if it does not exist. Acquires a write lock. |
| tryGetBuilderInfo(builderIndex) | BuilderInfo | O(1) | Retrieves the `BuilderInfo` wrapper for a given index from the cache. Returns null if not found. |
| getCachedBuilder(index, classType) | Builder<T> | O(1) | The primary method for game systems to retrieve a specific, type-checked builder instance. |
| validateBuilder(builderInfo) | boolean | O(D) | Recursively validates a builder and its entire dependency graph. D is dependency depth. |

## Integration Patterns

### Standard Usage
The intended use involves retrieving the singleton instance from the `NPCPlugin` or a service registry, then using it to load and access NPC configurations. Game logic should operate on integer indices, not string names.

```java
// During plugin initialization
BuilderManager manager = npcPlugin.getBuilderManager();
manager.registerFactory(new Role.RoleFactory());
// ... register other factories ...
boolean success = manager.loadBuilders(server.getAssetPack(), false);

// During gameplay (e.g., in a spawning system)
int npcIndex = manager.getIndex("skeletal_warrior");
if (npcIndex >= 0) {
    Builder<Role> roleBuilder = manager.tryGetCachedValidBuilder(npcIndex, Role.class);
    if (roleBuilder != null) {
        // Proceed to spawn NPC using the builder configuration
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderManager()`. The server relies on a single, shared instance to maintain a consistent state of all NPC assets. Always retrieve it from the central plugin or registry.
- **Late Registration:** Do not call `registerFactory` after `loadBuilders` has been invoked. Factories must be available before assets are parsed, otherwise the manager will not know how to construct the builder objects.
- **String-Based Lookups at Runtime:** Avoid calling `getIndex(name)` repeatedly in performance-critical code (e.g., every game tick). Resolve the name to an integer index once and cache the index.
- **Ignoring Validation State:** Do not use a builder without first checking that its `BuilderInfo` is valid. An invalid builder can cause severe runtime exceptions, crashes, or undefined behavior.

## Data Pipeline
The BuilderManager is the central processing unit in the NPC asset data flow. Data originates as raw text files and is transformed into validated, ready-to-use objects.

> Flow:
> JSON File on Disk -> `loadBuilders` / `AssetMonitor` -> `JsonParser` -> `BuilderFactory` -> **BuilderManager** (Validation & Dependency Resolution) -> **BuilderManager Cache** -> `getCachedBuilder` -> Game Systems (Spawner, AI)


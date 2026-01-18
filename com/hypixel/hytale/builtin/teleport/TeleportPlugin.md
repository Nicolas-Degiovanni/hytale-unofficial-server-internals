---
description: Architectural reference for TeleportPlugin
---

# TeleportPlugin

**Package:** com.hypixel.hytale.builtin.teleport
**Type:** Singleton

## Definition
```java
// Signature
public class TeleportPlugin extends JavaPlugin {
```

## Architecture & Concepts
The TeleportPlugin is a core server-side module that provides a comprehensive system for player and entity teleportation. It serves as the central authority for managing warp points, handling teleportation commands, and persisting location data. This class is not merely a collection of utilities; it is a foundational service that integrates deeply with several key engine systems:

-   **Plugin System:** It operates as a standard JavaPlugin, managed by the server's plugin lifecycle. Its singleton nature is enforced by this system.
-   **Command System:** It registers the canonical teleportation commands, including `/teleport`, `/warp`, and `/spawn`, making them available to players and server administrators.
-   **Entity Component System (ECS):** The plugin defines and manages two critical components:
    1.  **TeleportHistory:** Attached to player entities to track their previous locations, enabling features like a `/back` command.
    2.  **WarpComponent:** A tag component attached to in-world entities that represent a named warp point. This links a physical entity to a logical `Warp` data structure.
-   **Event Bus:** It subscribes to global engine events to drive its logic. For example, it listens for `AllWorldsLoadedEvent` to trigger the initial loading of warp data and `ChunkPreLoadProcessEvent` to dynamically spawn warp point entities as players explore the world.
-   **Persistence:** It is responsible for the serialization and deserialization of all warp points to and from the filesystem (`warps.json`), ensuring that defined locations persist across server restarts.
-   **World Map System:** It implements a `MarkerProvider` to publish the locations of all defined warps to the in-game world map, enhancing player navigation.

The static `get()` method provides global access, establishing the plugin as a centralized service for any other game system that needs to interact with teleportation logic or warp data.

### Lifecycle & Ownership
-   **Creation:** The TeleportPlugin is instantiated once by the server's PluginManager during the server bootstrap sequence. The static `instance` field is populated within the `setup` method, which is called by the plugin loader. Direct instantiation is not supported.
-   **Scope:** The instance is a session-scoped singleton. It persists for the entire lifetime of the server process.
-   **Destruction:** The `shutdown` method is invoked by the PluginManager when the server initiates a shutdown. The current implementation has no explicit cleanup logic, and the object is garbage collected upon process termination.

## Internal State & Concurrency
-   **State:** The class maintains significant mutable state, which represents the complete set of all defined warp points on the server.
    -   **warps:** A `ConcurrentHashMap` that stores all `Warp` objects, keyed by their lowercase ID. This is the primary data cache.
    -   **loaded:** An `AtomicBoolean` flag that indicates whether the `warps.json` file has been successfully parsed and loaded into memory. This acts as a gate to prevent access to warp data before it is ready.
    -   **saveLock:** A `ReentrantLock` used to serialize write operations to `warps.json`, preventing file corruption from concurrent save requests.
    -   **postSaveRedo:** An `AtomicBoolean` that works with `saveLock` to implement a non-blocking save pattern. If a save is requested while a write is already in progress, this flag is set to true, triggering another save operation once the lock is released.
-   **Thread Safety:** This class is designed to be thread-safe.
    -   The `warps` map uses a concurrent collection for safe atomic updates and reads from different threads.
    -   The `saveWarps` method employs a non-blocking `tryLock`. This is a critical performance consideration, as it prevents the main server thread or other critical threads from blocking on potentially slow disk I/O.
    -   Event handlers, such as `onChunkPreLoadProcess`, are often executed on world-specific worker threads. The implementation correctly marshals ECS operations (entity creation) back to the appropriate world's main thread via `world.execute`, adhering to the engine's single-threaded ECS modification rules.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | TeleportPlugin | O(1) | Statically retrieves the singleton instance of the plugin. |
| isWarpsLoaded() | boolean | O(1) | Returns true if the warp data has been loaded from disk. |
| loadWarps() | void | O(N) | Reads `warps.json` from the universe directory and populates the internal state. This is an I/O-bound operation. |
| saveWarps() | void | O(N) | Serializes the current warp state to `warps.json`. Uses a non-blocking lock to handle concurrent calls safely. |
| getWarps() | Map | O(1) | Returns a reference to the live map of all registered warps. |
| createWarp(warp, store) | Holder | O(1) | Factory method to construct a new entity representing a warp point. Does not spawn the entity in the world. |

## Integration Patterns

### Standard Usage
To interact with the teleportation system, other plugins must first retrieve the singleton instance and then access its public methods. It is crucial to verify that warps have been loaded before attempting to read them.

```java
// Example: Listing all available warps in another plugin
TeleportPlugin teleportPlugin = TeleportPlugin.get();

if (teleportPlugin.isWarpsLoaded()) {
    Map<String, Warp> allWarps = teleportPlugin.getWarps();
    for (Warp warp : allWarps.values()) {
        // Logic to process each warp
        System.out.println("Found warp: " + warp.getId());
    }
} else {
    // Handle the case where warps are not yet available
    System.err.println("Warp system is not ready.");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new TeleportPlugin()`. The plugin is managed by the server's lifecycle and must be accessed via the static `TeleportPlugin.get()` method.
-   **Premature Access:** Do not call `getWarps()` before the `AllWorldsLoadedEvent` has fired. The internal `warps` map will be empty. Always check with `isWarpsLoaded()` if operating outside of a lifecycle event that guarantees loading is complete.
-   **Unsynchronized State Modification:** Modifying the map returned by `getWarps()` is thread-safe at a collection level, but changes will be lost on restart unless `saveWarps()` is explicitly called. All logical operations that change the set of warps must be followed by a call to `saveWarps()`.

## Data Pipeline
The TeleportPlugin manages two primary data flows: loading warps from disk into the game world, and providing warp data to the UI map.

> **Flow 1: Warp Deserialization and Entity Spawning**
> Server Startup → `AllWorldsLoadedEvent` is fired → `TeleportPlugin.loadWarps()` → Filesystem Read (`warps.json`) → BSON Deserialization → **TeleportPlugin** (populates internal `warps` map) → Player Movement → `ChunkPreLoadProcessEvent` is fired → **TeleportPlugin** (`onChunkPreLoadProcess`) → `world.execute()` → `createWarp()` → New Warp Entity in World

> **Flow 2: World Map Marker Provisioning**
> Player opens map → `WarpMarkerProvider.update()` is called by WorldMapManager → **TeleportPlugin** (`get().getWarps()`) → Filter warps for current world → `WorldMapTracker.trySendMarker()` → Network Packet → Client UI Update


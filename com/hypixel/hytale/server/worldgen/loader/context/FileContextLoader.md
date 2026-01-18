---
description: Architectural reference for FileContextLoader
---

# FileContextLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.context
**Type:** Transient

## Definition
```java
// Signature
public class FileContextLoader {
```

## Architecture & Concepts
The FileContextLoader is a foundational component of the server's world generation pipeline. It acts as the primary orchestrator for discovering, parsing, and loading world structure definitions from the file system. Its core responsibility is to translate a specific, contract-driven directory structure—containing zones, biomes, and prefab configurations—into a unified, in-memory representation known as the FileLoadingContext.

This class embodies a "configuration-as-filesystem" pattern. The naming and organization of folders and files on disk directly dictate the composition of the world to be generated. For example, a directory named "EmeraldGrove" inside the "Zones" folder is treated as a distinct world zone.

The loader is not part of the continuous game loop. Instead, it is a critical part of the server's bootstrap or world-loading sequence. The FileLoadingContext it produces is an immutable snapshot of the world's configuration at load-time, which is then consumed by downstream services like the Biome Placement and Prefab Spawning systems.

## Lifecycle & Ownership
- **Creation:** An instance is created by a high-level world generation manager during the server's initialization phase. The creator must provide the root path to the world generation data and a Set of strings specifying which zones are required for the current world configuration.
- **Scope:** The object's lifetime is exceptionally short, scoped exclusively to a single invocation of the `load` method. It is a single-use object designed to perform one complete loading operation.
- **Destruction:** Once the `load` method returns the FileLoadingContext, the FileContextLoader instance has fulfilled its purpose and is eligible for garbage collection. There is no reason to maintain a reference to it post-load.

## Internal State & Concurrency
- **State:** The class is stateful, but its state is immutable after construction. The `dataFolder` and `zoneRequirement` fields are set once by the constructor and are not modified during the loading process. The `load` method is a pure function with respect to the class's internal state, producing a new object without side effects on the loader itself.
- **Thread Safety:** This class is **not thread-safe**. The `load` method performs numerous sequential file system I/O operations and assumes exclusive access to the data structures it builds. It is strictly intended for use in a single-threaded context, such as the main server thread during startup. Invoking `load` from multiple threads will lead to race conditions and unpredictable, likely corrupt, state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FileContextLoader(Path, Set) | constructor | O(1) | Constructs a new loader for a specific data directory and set of required zones. |
| load() | FileLoadingContext | O(F) | Scans the file system, parses all valid worldgen files, and returns a complete context. Throws a fatal `Error` if validation fails or I/O exceptions occur. F = total files and directories scanned. |

## Integration Patterns

### Standard Usage
The loader is instantiated, used immediately to load the context, and then discarded. The resulting context object is passed to other services.

```java
// In a world generation service during server startup
Path worldgenRoot = server.getDataPath().resolve("worldgen");
Set<String> requiredZones = Set.of("EmeraldGrove", "Borea");

try {
    FileContextLoader loader = new FileContextLoader(worldgenRoot, requiredZones);
    FileLoadingContext worldDataContext = loader.load();

    // Pass the fully loaded context to other systems
    this.biomeManager.initialize(worldDataContext);
    this.prefabService.initialize(worldDataContext);

} catch (Error e) {
    // A thrown Error indicates a catastrophic, unrecoverable configuration problem.
    // The server should not attempt to continue.
    HytaleLogger.getLogger().at(Level.SEVERE).withCause(e).log("Failed to load world generation context. Server startup aborted.");
    // Terminate server startup
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache and reuse a FileContextLoader instance. It is designed as a short-lived, single-use object.
- **Concurrent Invocation:** Never call the `load` method from multiple threads. The underlying file system scanning and context building are not atomic or thread-safe.
- **Ignoring Errors:** The `load` method intentionally throws a raw `Error` on critical failures. This signals a non-recoverable state, such as a missing required zone or a malformed file. Code must not swallow this `Error`; it should be caught at a high level to safely terminate the server startup process.

## Data Pipeline
The FileContextLoader is the entry point for transforming on-disk assets into a usable in-memory data structure for the world generator.

> Flow:
> File System (Folders, .json files) -> Java NIO `Files.find` -> **FileContextLoader** -> FileLoadingContext (Object Graph) -> World Generation Services (e.g., BiomeManager)


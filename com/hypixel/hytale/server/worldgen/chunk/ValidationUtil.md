---
description: Architectural reference for ValidationUtil
---

# ValidationUtil

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Utility

## Definition
```java
// Signature
public class ValidationUtil {
```

## Architecture & Concepts
ValidationUtil is a stateless, static utility class designed to perform deep, pre-emptive validation of the entire world generation configuration. Its primary role is to act as a "fail-fast" mechanism during server initialization. Before the server commits to using a world generation profile, this utility traverses the complete hierarchy of zones, biomes, caves, and unique structures to verify that every single referenced prefab asset is loadable from the filesystem.

This process is critical for server stability. A missing or corrupt prefab file would otherwise cause a runtime exception during live chunk generation, potentially corrupting world data or crashing the server. By validating the entire prefab dependency graph upfront, ValidationUtil guarantees that the configuration is structurally sound and all required assets are present.

The validation is executed asynchronously via a CompletableFuture, allowing the server startup sequence to proceed with other tasks while this potentially I/O-heavy operation completes. Errors are not thrown but are instead logged verbosely with a full trace of the configuration path, and a final boolean result indicates the overall success or failure.

## Lifecycle & Ownership
- **Creation:** As a static utility class, ValidationUtil is never instantiated. Its methods are invoked directly on the class.
- **Scope:** The class and its methods are available for the entire lifetime of the server process. The operational scope of a validation run is confined to a single, asynchronous task.
- **Destruction:** Not applicable. No state is held, and no cleanup is required.

## Internal State & Concurrency
- **State:** ValidationUtil is entirely stateless. All data required for a validation run, such as the traversal path (trace) and visited nodes (encounteredNodes), is passed as method parameters and confined to the stack of the asynchronous task. It holds no instance or static fields.
- **Thread Safety:** The public API method, isInvalid, is thread-safe. It accepts an Executor and safely offloads the entire validation process to it. The internal helper methods (*isZoneInvalid*, *isBiomeInvalid*, etc.) are **not** thread-safe and are not designed for concurrent access. They rely on being executed sequentially within the single thread provided by the Executor to safely mutate their local state (e.g., the trace Deque). This design encapsulates the complex, stateful traversal logic within a safe, single-threaded asynchronous boundary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isInvalid(zonePatternProvider, executor) | boolean | O(P * D) | Initiates a full, asynchronous validation of the worldgen configuration. Returns true if any prefab fails to load. Complexity is I/O bound, proportional to the total number of Prefabs (P) and their nesting Depth (D). |

## Integration Patterns

### Standard Usage
This utility should be invoked once during the server startup sequence after the world generation configuration has been loaded into memory. The result must be checked to determine if the server should abort its launch.

```java
// During server initialization
ZonePatternProvider provider = loadWorldGenConfig();
Executor validationExecutor = Executors.newSingleThreadExecutor();

boolean hasErrors = ValidationUtil.isInvalid(provider, validationExecutor);

if (hasErrors) {
    // Abort server startup
    HytaleLogger.getLogger().severe("World generation validation failed. Server cannot start.");
    System.exit(1);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Result:** Calling isInvalid and failing to act on a `true` return value completely negates the purpose of this utility. This will lead to predictable runtime crashes.
- **Using a Direct Executor:** Supplying a direct or same-thread executor will cause this I/O-intensive operation to block the calling thread. If called from the main server thread, this will freeze the server during startup.
- **Calling Private Helpers:** The internal recursive methods are not part of the public contract and are not thread-safe. Invoking them directly will lead to unpredictable behavior and race conditions.

## Data Pipeline
ValidationUtil processes a complex object graph representing the world configuration and reduces it to a single boolean value, producing detailed logs as a side effect in case of failure.

> Flow:
> ZonePatternProvider (In-Memory Config) -> **ValidationUtil.isInvalid()** -> CompletableFuture on Executor -> Recursive Traversal of Zones, Biomes, Caves -> WorldGenPrefabLoader -> Filesystem I/O -> Boolean Result & HytaleLogger Error Output


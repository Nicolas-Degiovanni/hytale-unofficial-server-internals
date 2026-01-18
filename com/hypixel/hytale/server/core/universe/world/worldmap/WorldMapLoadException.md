---
description: Architectural reference for WorldMapLoadException
---

# WorldMapLoadException

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap
**Type:** Transient Model

## Definition
```java
// Signature
public class WorldMapLoadException extends Exception {
```

## Architecture & Concepts
WorldMapLoadException is a specialized, checked exception that signals a critical failure during the loading or deserialization of a server's world map data. It is a fundamental component of the server's error handling and world persistence subsystem.

Architecturally, its existence as a checked exception enforces a strict contract on any system that interacts with world data. Callers are syntactically required to anticipate and handle potential loading failures, preventing entire classes of unhandled errors that could lead to server corruption or instability.

This class standardizes error reporting for a complex process. By wrapping lower-level exceptions (like an IOException from disk reads or a DeserializationException from data parsing), it provides a single, high-level failure type for the rest of the server to reason about. The inclusion of the `getTraceMessage` method, which delegates to ExceptionUtil, indicates a design pattern for creating consistent, detailed, and multi-layered error logs across the entire Hytale codebase.

## Lifecycle & Ownership
- **Creation:** An instance is created and thrown exclusively by components responsible for reading and interpreting world map data from a persistent source (e.g., disk, database). This typically occurs deep within a world loading service when a file is not found, is corrupt, or is in an invalid format.
- **Scope:** The object's lifetime is ephemeral. It exists only for the duration of its propagation up the call stack.
- **Destruction:** It is eligible for garbage collection as soon as it is caught and handled by an error-handling mechanism, such as a top-level exception logger in the server's main loop or a world management service. It holds no persistent state and is not meant to be stored.

## Internal State & Concurrency
- **State:** The state of a WorldMapLoadException is **effectively immutable**. Its core data—the message and the causal exception chain—is set at construction time via its `super` call to `java.lang.Exception` and cannot be modified thereafter.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, an instance can be safely passed across thread boundaries without risk of data corruption. However, its typical use case is confined to the propagation chain within a single thread.

## API Surface
The public API is minimal, focusing on a standardized way to extract a comprehensive diagnostic message.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldMapLoadException(String) | constructor | O(1) | Constructs an exception with a detail message. |
| WorldMapLoadException(String, Throwable) | constructor | O(1) | Constructs an exception with a message and a root cause. |
| getTraceMessage() | String | O(N) | Generates a formatted, chained message of all causes. N is the depth of the cause chain. |
| getTraceMessage(String) | String | O(N) | Generates a formatted, chained message using a custom delimiter. |

## Integration Patterns

### Standard Usage
The intended use is within a `try-catch` block where world loading operations are performed. The exception should be caught specifically to allow for targeted error handling, such as aborting a world startup sequence and logging the detailed trace message.

```java
// A WorldManager attempts to load a world.
try {
    world.loadMapData(worldId);
} catch (WorldMapLoadException e) {
    // Log the specific, detailed error for diagnostics.
    // The trace message provides a full view of the failure chain.
    ServerLog.error("Failed to load world map for '{}': {}", worldId, e.getTraceMessage());
    // Trigger fallback logic, like loading a backup or shutting down.
    server.abortWorldLoad(worldId, "Map data is unreadable.");
}
```

### Anti-Patterns (Do NOT do this)
- **Catching Generic Exception:** Avoid catching `java.lang.Exception` when this specific type is thrown. Doing so discards valuable contextual information and leads to vague error handling.
- **Swallowing the Exception:** Never catch a WorldMapLoadException and ignore it. An empty catch block is a critical defect, as it hides a fundamental server failure that will almost certainly cause cascading problems.
- **Omitting the Cause:** When catching a lower-level exception (e.g., IOException) and wrapping it, always pass the original exception as the `cause` parameter. Failure to do so destroys the stack trace and makes debugging nearly impossible.

## Data Pipeline
WorldMapLoadException functions as a terminal signal in a data pipeline. It represents the failure state of a data flow, not a successful transformation.

> Flow:
> Disk Read Request -> I/O Subsystem -> **[FAILURE: IOException]** -> World Data Parser -> **[CAUGHT & WRAPPED]** -> **WorldMapLoadException Thrown** -> Propagates to World Manager -> Server State Controller -> **[HANDLED]** -> Error Log & Server Shutdown/Fallback


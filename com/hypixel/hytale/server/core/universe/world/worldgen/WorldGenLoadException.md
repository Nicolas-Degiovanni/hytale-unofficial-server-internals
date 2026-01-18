---
description: Architectural reference for WorldGenLoadException
---

# WorldGenLoadException

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenLoadException extends Exception {
```

## Architecture & Concepts
WorldGenLoadException is a specialized, checked exception that signals a fatal error during the loading or initialization phase of the server's world generation system. Its primary architectural role is to provide a distinct failure type, allowing higher-level systems like the WorldManager or the server bootstrap process to catch and handle world generation failures specifically, differentiating them from generic IO or runtime errors.

By extending Exception, it forces callers of world generation loading methods to explicitly handle potential failures, ensuring that a corrupt or misconfigured world cannot be silently ignored. The class encapsulates not only an error message but also the underlying cause, providing a complete diagnostic context for server administrators.

## Lifecycle & Ownership
- **Creation:** Instantiated and thrown by components within the world generation pipeline when an unrecoverable state is detected. Common triggers include parsing errors in world configuration files, missing generator assets, or validation failures.
- **Scope:** The object's lifetime is ephemeral. It exists only for the duration of its propagation up the call stack until it is caught by an appropriate exception handler.
- **Destruction:** Once caught and processed (e.g., logged), the exception object is no longer referenced and becomes eligible for garbage collection. It holds no persistent state and is not managed by any container.

## Internal State & Concurrency
- **State:** The state of a WorldGenLoadException is effectively immutable after construction. It consists of a detail message and an optional causal chain (the `cause` Throwable), both of which are set once in the constructor.
- **Thread Safety:** This class is inherently thread-safe. Exception objects are typically confined to the stack of a single thread. Even if a reference were shared, its immutable nature prevents data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldGenLoadException(message) | constructor | O(1) | Creates an exception with a specific detail message. |
| WorldGenLoadException(message, cause) | constructor | O(1) | Creates an exception that wraps an underlying cause. |
| getTraceMessage() | String | O(N) | Traverses the entire causal chain to build a concatenated, human-readable error message. N is the depth of the exception chain. |
| getTraceMessage(joiner) | String | O(N) | A variant of getTraceMessage that allows a custom separator between chained messages. |

## Integration Patterns

### Standard Usage
This exception should be caught by the system responsible for orchestrating world loading. The handler should treat the failure as critical, log the full trace for debugging, and typically prevent the world from being loaded.

```java
// A WorldManager or similar service catching the specific failure
try {
    worldGenerator.loadFromDisk(worldId);
} catch (WorldGenLoadException e) {
    // Log the detailed, combined message for server operators
    log.fatal("Failed to load world generation data: " + e.getTraceMessage(), e);
    // Trigger a server shutdown or unload the failed world
    server.shutdown("Unrecoverable world generation error.");
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing the Exception:** Never catch WorldGenLoadException and ignore it. It signals a fundamental problem with the world's integrity that will cause severe issues if the server continues to run.
- **Generic Catch:** Avoid catching a generic Exception when a WorldGenLoadException is expected. Doing so loses the specific context and makes error handling less precise.
- **Control Flow:** Do not use this exception for non-exceptional control flow. It is for signalling catastrophic failures only.

## Data Pipeline
WorldGenLoadException acts as a terminal failure state within the world loading data pipeline. It diverts the normal flow into an error handling and shutdown path.

> Flow:
> World Configuration File -> Parser -> **WorldGenLoadException (on error)** -> WorldManager Catch Block -> Server Log -> Server Shutdown


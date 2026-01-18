---
description: Architectural reference for WorldLoadCancelledException
---

# WorldLoadCancelledException

**Package:** com.hypixel.hytale.server.core.universe
**Type:** Signal Exception

## Definition
```java
// Signature
public class WorldLoadCancelledException extends RuntimeException {
```

## Architecture & Concepts
WorldLoadCancelledException is a specialized, unchecked exception used as a control-flow signal within the server's world loading process. Its primary architectural role is to provide a clean, type-safe mechanism for aborting a complex, potentially long-running world loading operation.

By extending RuntimeException, it signals a non-recoverable state change that should not be mandatorily caught by intermediate layers. This allows the exception to propagate cleanly up the call stack from a deeply nested loading task to the high-level UniverseManager or server bootstrap logic responsible for orchestrating the load. This avoids polluting method signatures with checked exceptions and simplifies the cancellation logic.

It serves as a formal contract between the world loading task and its orchestrator, distinguishing a deliberate cancellation from other unexpected failures like disk I/O errors or data corruption.

## Lifecycle & Ownership
- **Creation:** Instantiated and thrown by a world loading task when it receives an external cancellation signal. This typically occurs when a user issues a stop command or the server initiates a shutdown while a world is still being prepared.
- **Scope:** Ephemeral. The object exists only for the duration of stack unwinding, from the point it is thrown to the point it is caught.
- **Destruction:** The exception object is eligible for garbage collection immediately after its corresponding catch block completes execution. It holds no persistent state.

## Internal State & Concurrency
- **State:** Inherently immutable. Like all exceptions, its internal state (primarily the stack trace) is captured at the moment of instantiation and is not modified thereafter. It carries no custom state beyond what is provided by the base RuntimeException.
- **Thread Safety:** Not applicable. Exceptions are thread-local constructs. An instance of WorldLoadCancelledException is created, thrown, and caught entirely within the context of a single thread's execution stack. It is never shared across threads.

## API Surface
This class adds no new public methods and is not intended to be interacted with beyond its construction and type. The standard `java.lang.Throwable` interface is its only API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (constructor) | WorldLoadCancelledException | O(1) | Creates a new instance. Typically used with a throw statement. |

## Integration Patterns

### Standard Usage
The sole intended use is to wrap world-loading logic in a try-catch block, specifically listening for this exception type to handle graceful shutdown of the loading sequence.

```java
// Correctly handling a deliberate world load cancellation
try {
    worldLoader.loadWorld(worldData);
} catch (WorldLoadCancelledException e) {
    // This is an expected, clean abort.
    // Log the cancellation and perform necessary cleanup.
    log.info("World loading was cancelled as requested.");
    server.transitionToState(ServerState.STOPPED);
} catch (Exception e) {
    // An unexpected error occurred.
    log.error("A critical error occurred during world load!", e);
    server.transitionToState(ServerState.CRASHED);
}
```

### Anti-Patterns (Do NOT do this)
- **Generic Catching:** Do not catch the generic Exception or RuntimeException to handle cancellation. This conflates a controlled abort with a genuine, unexpected error, leading to incorrect server state transitions and logging.
- **Swallowing:** Never catch this exception and ignore it. Doing so will leave the server in an indeterminate, partially-loaded state, as the orchestrator will not be notified that the operation failed to complete.
- **Misuse for Other Errors:** Do not throw this exception for failures unrelated to cancellation, such as a missing world file or corrupted chunk data. Use more specific exceptions (e.g., WorldNotFoundException, ChunkCorruptionException) for such cases.

## Data Pipeline
This class does not participate in a data pipeline. It represents a break in a control-flow pipeline.

> Flow:
> External Cancellation Request -> World Loading Thread -> **throw new WorldLoadCancelledException()** -> UniverseManager Catch Block -> Server State Transition (e.g., to STOPPED)


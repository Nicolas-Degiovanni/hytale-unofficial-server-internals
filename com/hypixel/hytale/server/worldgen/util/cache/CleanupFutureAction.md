---
description: Architectural reference for CleanupFutureAction
---

# CleanupFutureAction

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Utility

## Definition
```java
// Signature
public class CleanupFutureAction implements Runnable {
```

## Architecture & Concepts
The CleanupFutureAction class is a specialized, single-purpose utility that acts as a bridge between the Java Garbage Collector (GC) and the Concurrency framework. It leverages the `java.lang.ref.Cleaner` API, a modern and reliable replacement for finalization, to ensure resource cleanup.

Its primary role is to prevent resource leaks within asynchronous systems, particularly caches. In a typical scenario, a system might schedule a future task (e.g., to expire a cache entry after a timeout) and associate it with an object (the cache entry itself). If the object is evicted or otherwise becomes unreachable and is garbage collected, the scheduled task becomes redundant. Without proper cleanup, this orphaned task would needlessly occupy resources in the `ScheduledExecutorService` until its delay expires.

CleanupFutureAction solves this by creating a `Runnable` whose sole responsibility is to cancel a `ScheduledFuture`. This action is registered with the static `CLEANER` instance, linking it to the lifecycle of the owner object. When the owner object is garbage collected, the `Cleaner` automatically executes this action, ensuring the associated future is promptly cancelled.

### Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level component, typically a cache manager or a system that pairs transient objects with scheduled tasks. It is immediately registered with the static `CLEANER` along with the object whose lifecycle is being monitored.
- **Scope:** The lifecycle of a CleanupFutureAction instance is managed entirely by the `java.lang.ref.Cleaner`. It is held by the `Cleaner` from the moment of registration until the associated owner object becomes phantom-reachable and the action is executed.
- **Destruction:** The action is invoked on a dedicated, high-priority daemon thread managed by the `Cleaner`. After its `run` method completes, the CleanupFutureAction object itself becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The class is effectively immutable. Its internal state consists of a single final field, `future`, which is set at construction time and never modified.
- **Thread Safety:** This class is inherently thread-safe. The `run` method is guaranteed to be called by the `Cleaner`'s dedicated thread. The underlying `ScheduledFuture.cancel` method is specified to be thread-safe, making the entire operation safe for concurrent environments. No external locking is required.

## API Surface
The public contract is minimal, consisting only of the constructor. The `run` method is part of the `Runnable` interface and is not intended for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CleanupFutureAction(future) | constructor | O(1) | Creates a new cleanup action tied to the provided future. |
| run() | void | O(1) | Implements the Runnable interface. Cancels the encapsulated future. **Warning:** This method is for internal use by the Cleaner service only. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly in most application logic. It is a low-level utility integrated into components that manage object lifecycles and associated asynchronous tasks. The following demonstrates the conceptual pattern.

```java
// Conceptual Usage within a Cache System
// Assume 'executor' is a ScheduledExecutorService

// 1. An object is created, e.g., a cache entry for world data.
Object worldChunkData = new Object();

// 2. A task is scheduled, associated with the object's lifecycle.
ScheduledFuture<?> expiryTask = executor.schedule(() -> {
    // Logic to expire the chunk data
}, 10, TimeUnit.MINUTES);

// 3. The cleanup action is registered with the Cleaner.
// When 'worldChunkData' is garbage collected, the 'run' method of the
// CleanupFutureAction will be invoked, cancelling the 'expiryTask'.
CleanupFutureAction.CLEANER.register(worldChunkData, new CleanupFutureAction(expiryTask));
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never instantiate and call `run` directly. This completely bypasses the garbage collection trigger and serves only to cancel the future immediately, defeating the purpose of the class.
    ```java
    // ANTI-PATTERN: This is incorrect and serves no purpose.
    new CleanupFutureAction(myFuture).run();
    ```
- **Long-Lived Owners:** Do not register a cleanup action against an object that will never be garbage collected, such as an object held in a static field or a singleton. The cleanup action will never be triggered, resulting in a memory leak as the `Cleaner` will hold a reference to the action indefinitely.

## Data Pipeline
This component does not process data. Instead, it operates on a signal-based flow triggered by the JVM's garbage collector.

> Flow:
> Owner Object becomes phantom-reachable -> JVM Garbage Collector -> **java.lang.ref.Cleaner** -> Invokes **CleanupFutureAction.run()** -> ScheduledFuture.cancel() -> Task is de-scheduled from ExecutorService


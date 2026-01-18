---
description: Architectural reference for DirectoryHandlerChangeTask
---

# DirectoryHandlerChangeTask

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Transient

## Definition
```java
// Signature
public class DirectoryHandlerChangeTask implements Runnable {
```

## Architecture & Concepts

The DirectoryHandlerChangeTask is a stateful, self-scheduling job that acts as a temporal accumulator and processor for raw file system events. It is a fundamental component of the AssetMonitor subsystem, designed to solve the problem of "event noise" from underlying file watchers.

Operating systems often emit a rapid, chaotic sequence of events for a single logical file operation (e.g., saving a file might trigger separate MODIFY and CREATE events). This class captures these raw events over a short period, debounces them, sorts them chronologically, and groups them into logical batches. These clean, ordered batches are then dispatched to a registered AssetMonitorHandler for final processing, such as reloading a game asset.

Its core design principles are:
*   **Accumulation:** It collects file system events in an internal map for a configured delay period.
*   **Debouncing:** By waiting for a quiet period, it avoids reacting to every intermediate event during a rapid sequence of file modifications.
*   **Batching:** It intelligently groups related events before dispatching them, allowing handlers to perform more efficient bulk operations.
*   **Self-Management:** The task schedules its own execution and automatically cancels itself when it becomes idle, minimizing system overhead.

This class is not intended for direct use by developers; it is an internal implementation detail managed exclusively by the AssetMonitor.

## Lifecycle & Ownership

The lifecycle of a DirectoryHandlerChangeTask is ephemeral and tightly controlled by its parent AssetMonitor.

*   **Creation:** An instance is created by the AssetMonitor when the first file system event is detected for a directory that has a registered AssetMonitorHandler. A single task is responsible for all events within a single parent directory.
*   **Scope:** The object persists as long as new file system events are being added to it. Upon creation, it schedules itself to run after a 1000ms delay. Each new event added to the task effectively resets this "idle" timer.
*   **Destruction:** The task is designed to clean itself up automatically. It is destroyed and garbage collected under two conditions:
    1.  **Idleness:** The scheduled `run` method executes, and it determines that no new events have been added since its last check. In this case, it cancels its own schedule and is removed from the AssetMonitor's active task list.
    2.  **Explicit Cancellation:** The `cancelSchedule` method is called, typically by the AssetMonitor during a system shutdown or when a directory watch is explicitly removed.

## Internal State & Concurrency

*   **State:** This class is highly mutable. Its primary state is the `paths` map, which stores incoming PathEvent objects keyed by their file Path. It also maintains an `AtomicBoolean` named `changed` as a dirty flag to coordinate between threads.

*   **Thread Safety:** This class is **not** inherently thread-safe and relies on a strict concurrency model enforced by the AssetMonitor.
    *   The `addPath` and `removePath` methods are expected to be called from a single thread—the AssetMonitor's file watcher thread. These methods perform non-atomic writes to the internal `paths` map.
    *   The `run` method is executed on a separate scheduler thread pool.
    *   Coordination is achieved via the `changed` AtomicBoolean. It ensures that writes from the watcher thread are visible to the scheduler thread. The `run` method's first action is to atomically read and reset this flag (`getAndSet(false)`), which also acts as a memory barrier.

    **WARNING:** The design assumes that the `paths` map is not modified by the watcher thread *during* the brief window that the `run` method is copying its contents. While the window is small, this pattern is fragile. Direct multi-threaded access to an instance of this class without external locking will lead to unpredictable behavior and data corruption.

## API Surface

The public API is intended for internal use by the AssetMonitor system only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run() | void | O(N log N) | The scheduled entry point. Processes accumulated events. Complexity is dominated by sorting the N events. |
| addPath(path, pathEvent) | void | O(1) | Adds a file system event to the internal queue and marks the task as changed. |
| removePath(path) | void | O(1) | Removes a pending event. If the queue becomes empty, the task is cancelled. |
| markChanged() | void | O(1) | Manually flags the task as needing to run, ensuring its next scheduled execution will process events. |
| cancelSchedule() | void | O(1) | Immediately cancels the scheduled task and unregisters it from the parent AssetMonitor. |

## Integration Patterns

### Standard Usage

This class should never be instantiated or manipulated directly. The correct pattern is to interact with its owner, the AssetMonitor, which abstracts away the creation and management of these tasks.

```java
// Conceptual Example: The AssetMonitor manages the task internally.
// You do NOT write this code; this is what happens under the hood.

// 1. A handler is registered for a directory.
AssetMonitor monitor = context.getService(AssetMonitor.class);
monitor.register("/opt/hytale/assets/models", (events) -> {
    // ... process clean, batched events ...
});

// 2. A file event occurs. The monitor finds or creates the task.
DirectoryHandlerChangeTask task = monitor.getOrCreateTaskFor("/opt/hytale/assets/models");

// 3. The monitor feeds the raw event to the task.
task.addPath(newPath, newEvent);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not call `new DirectoryHandlerChangeTask()`. This creates an orphan task that is not managed by the AssetMonitor's lifecycle or scheduler, resulting in a memory leak.
*   **Directly Calling run:** Do not invoke the `run` method manually. This bypasses the critical time-based debouncing logic and executes the processing logic on the wrong thread, violating the class's concurrency model.
*   **Multi-threaded Access:** Do not call `addPath` from multiple threads simultaneously. The internal `paths` map is not thread-safe and will become corrupted.

## Data Pipeline

The DirectoryHandlerChangeTask is a key stage in the asset monitoring pipeline, transforming a high-frequency stream of raw events into a low-frequency stream of logical batches.

> Flow:
> OS File System Event -> Java WatchService -> AssetMonitor -> **DirectoryHandlerChangeTask** (Accumulate & Batch) -> AssetMonitorHandler (Process) -> Asset Reload System


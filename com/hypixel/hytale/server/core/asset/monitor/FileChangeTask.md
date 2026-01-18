---
description: Architectural reference for FileChangeTask
---

# FileChangeTask

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Transient

## Definition
```java
// Signature
public class FileChangeTask implements Runnable {
```

## Architecture & Concepts
The FileChangeTask is a short-lived, stateful object that acts as a debouncing and stabilization mechanism for file system events. It is a core component of the AssetMonitor system, designed to solve the "incomplete write" problem where a single file modification can trigger numerous, rapid-fire file system notifications.

This class is not a generic task; it is purpose-built for asset monitoring. When the AssetMonitor detects a change to a file on disk (e.g., an `ENTRY_CREATE` or `ENTRY_MODIFY` event), it does not process the change immediately. Instead, it instantiates a FileChangeTask. This task schedules itself to run after a short delay (200ms).

The primary function of this delay is to wait for file I/O to stabilize. When the task finally executes its `run` method, it performs a final check on the file's size. If the file is still growing, the task aborts, assuming the write is still in progress. The system then relies on the operating system to send another modification event, which will start a new debouncing cycle. If the file size is stable, the task signals the AssetMonitor to proceed with the actual asset reload.

This pattern ensures that the asset system only ever attempts to load files that are in a complete and stable state, preventing I/O exceptions and the loading of corrupt or partial data.

## Lifecycle & Ownership
-   **Creation:** A FileChangeTask is instantiated exclusively by the AssetMonitor when its internal `WatchService` reports a file system event. The AssetMonitor is the sole owner and manager of these tasks.

-   **Scope:** The object's lifetime is extremely short, typically lasting just over 200 milliseconds. It represents a single, pending file-change operation. If a new event for the same file path occurs before the 200ms delay has elapsed, the existing FileChangeTask is cancelled and replaced by a new one, effectively resetting the debounce timer.

-   **Destruction:** A task is destroyed under one of the following conditions:
    1.  **Normal Completion:** The task executes after its delay, finds the file is stable, and notifies the AssetMonitor. It then cleans up its own schedule and is dereferenced.
    2.  **Superseded:** A newer file event for the same path causes the AssetMonitor to explicitly call `cancelSchedule` on the existing task before creating a new one.
    3.  **Self-Termination:** The task executes but finds the target file has been deleted or is still being written to. In this case, it cancels its own schedule and terminates without notifying the AssetMonitor to proceed.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains the file path, the event that triggered it, the last known file size (`lastSize`), and a reference to its own `ScheduledFuture`. This state is captured at creation and is essential for the debouncing logic within the `run` method.

-   **Thread Safety:** A FileChangeTask instance is not thread-safe and is not designed to be. Its lifecycle is strictly controlled. It is created on the AssetMonitor's file-watcher thread, and its `run` method is executed by a separate scheduler thread. All state access is sequential and confined to this lifecycle, eliminating the need for explicit locking within the class itself. Synchronization is the responsibility of the managing AssetMonitor.

## API Surface
The public API is minimal, as this class is intended for internal use by the AssetMonitor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run() | void | O(1) | Executes the delayed file check. This is the core logic and should only be invoked by a scheduler. |
| cancelSchedule() | void | O(1) | Immediately cancels the scheduled task and removes its tracking reference from the AssetMonitor. |

## Integration Patterns

### Standard Usage
This class is an internal component of the AssetMonitor and should not be used directly by other systems. The following conceptual example illustrates how the AssetMonitor manages it.

```java
// Conceptual code within AssetMonitor
// A map 'runningTasks' tracks tasks by Path.

Path eventPath = event.context();
FileChangeTask existingTask = runningTasks.get(eventPath);

// If a task for this path already exists, cancel and replace it.
if (existingTask != null) {
    existingTask.cancelSchedule();
}

try {
    // Create a new task, which self-schedules.
    FileChangeTask newTask = new FileChangeTask(this, eventPath, event);
    runningTasks.put(eventPath, newTask);
} catch (IOException e) {
    LOGGER.at(Level.WARNING).withCause(e).log("Failed to create file change task for %s", eventPath);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new FileChangeTask()`. The class is tightly coupled to the AssetMonitor's lifecycle and scheduler. Instantiating it outside of this context will lead to memory leaks and non-functional behavior.
-   **Calling run() Directly:** Manually invoking the `run` method bypasses the critical 200ms debouncing delay, defeating the entire purpose of the class and risking the load of incomplete asset files.
-   **Reusing Instances:** A FileChangeTask is a one-shot object designed for a single event. It cannot be reset or reused.

## Data Pipeline
The FileChangeTask acts as a critical delay and validation gate in the asset hot-reloading data flow.

> Flow:
> OS File System Event -> Java NIO WatchService -> AssetMonitor -> **FileChangeTask (Creation & 200ms Delay)** -> Scheduler Thread Pool -> **FileChangeTask (run() executes size check)** -> AssetMonitor.onDelayedChange() -> Asset Reloading Subsystem


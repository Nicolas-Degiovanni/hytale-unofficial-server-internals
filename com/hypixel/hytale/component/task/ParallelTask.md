---
description: Architectural reference for ParallelTask
---

# ParallelTask

**Package:** com.hypixel.hytale.component.task
**Type:** Stateful Task

## Definition
```java
// Signature
public class ParallelTask<D extends IntConsumer> extends CountedCompleter<Void> {
```

## Architecture & Concepts
The ParallelTask class is a high-level orchestrator for massively parallel, "divide and conquer" style workloads. It is built directly upon the Java Fork/Join framework, specifically using CountedCompleter to provide a more efficient and manageable abstraction than raw ForkJoinTask implementations.

Its primary architectural role is to partition a large set of work into discrete sub-tasks, represented by ParallelRangeTask, and execute them concurrently across a ForkJoinPool. This component is fundamental for performance-critical systems like world generation, chunk processing, or large-scale entity updates where work can be performed independently on different data sets.

The design employs a factory pattern via the `Supplier<D>` generic parameter. This ensures that each worker thread receives a unique instance of the worker logic (an IntConsumer), preventing state-related race conditions and simplifying the implementation of the worker itself. The ParallelTask manages the execution graph, while the supplied IntConsumer defines the actual work to be done.

## Lifecycle & Ownership
The lifecycle of a ParallelTask instance is ephemeral and manually controlled. It is designed to be configured, executed, and then either discarded or re-initialized for another run.

-   **Creation:** An instance is created directly via its constructor (`new ParallelTask(...)`) by a system that needs to execute a parallel workload. It is not a managed service and should not be treated as a long-lived singleton.

-   **Scope:** The object's scope is typically confined to a single method or a single frame update. Its lifecycle follows a strict three-phase process:
    1.  **Configuration:** After creation, `init()` is called. The workload is then defined by calling `appendTask()` one or more times. During this phase, the task is dormant.
    2.  **Execution:** The `doInvoke()` method is called. This submits the task to the ForkJoinPool and **blocks the calling thread** until all sub-tasks have completed. The task is considered "running" during this phase.
    3.  **Completion:** Once `doInvoke()` returns, the task is complete. It can be safely discarded or prepared for a new workload by calling `init()` again.

-   **Destruction:** The object is eligible for garbage collection as soon as it goes out of scope. There are no explicit cleanup or `close()` methods.

## Internal State & Concurrency
-   **State:** ParallelTask is a highly stateful and mutable object. Its internal state includes the array of subTasks, the current number of tasks (`size`), and the `running` execution flag. This state is valid only for a single execution cycle and must be reset via `init()` for reuse.

-   **Thread Safety:** This class has a complex thread-safety profile that is critical to understand.
    -   **Configuration methods (`init`, `appendTask`) are NOT thread-safe.** They must be called from the single, controlling thread that orchestrates the work, before execution begins.
    -   **Execution methods (`compute`, `doInvoke`) are designed for a concurrent environment.** The `compute` method will be executed by multiple threads from a ForkJoinPool. The `doInvoke` method safely manages the transition into and out of the running state.
    -   The `running` field is `volatile` to ensure that changes to the task's execution state are immediately visible to all threads, preventing illegal state transitions such as appending a new task while it is already executing.

## API Surface
The public API is designed to support the strict configuration-then-execution lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init() | void | O(1) | Resets the task, clearing all sub-tasks and preparing it for a new workload. Throws IllegalStateException if the task is running. |
| appendTask() | ParallelRangeTask | O(1) amortized | Adds a new sub-task to the workload. The returned object must be configured before execution. Throws IllegalStateException if the task is running. |
| doInvoke() | void | O(N) | Submits the entire task graph for execution and blocks the current thread until all work is complete. N is the total work across all sub-tasks. |
| size() | int | O(1) | Returns the number of sub-tasks currently configured. |
| get(int i) | ParallelRangeTask | O(1) | Retrieves a previously appended sub-task by its index. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves initializing the task, appending and configuring all sub-tasks, and finally invoking it. This pattern is ideal for batch processing within a single game tick.

```java
// 1. Create a supplier for the worker logic. This will be called for each sub-task.
Supplier<MyIntConsumer> workerSupplier = () -> new MyIntConsumer(world);

// 2. Instantiate and initialize the main task.
ParallelTask<MyIntConsumer> task = new ParallelTask<>(workerSupplier);
task.init();

// 3. Configure the sub-tasks.
for (int i = 0; i < 16; i++) {
    ParallelRangeTask<MyIntConsumer> subTask = task.appendTask();
    subTask.setRange(i * 100, (i + 1) * 100);
}

// 4. Execute the entire workload and wait for completion.
// WARNING: This is a blocking call.
task.doInvoke();

// The task can now be discarded or re-used by calling task.init() again.
```

### Anti-Patterns (Do NOT do this)
-   **Reusing without Re-initialization:** Calling `doInvoke()` multiple times without calling `init()` in between will result in an exception, as the underlying ForkJoinTask cannot be restarted.
-   **Modification During Execution:** Attempting to call `appendTask()` or `init()` from another thread while `doInvoke()` is in progress is an error. The internal `running` flag will throw an IllegalStateException to prevent this.
-   **Sharing Worker State:** The `Supplier` should always create a *new* instance of the worker if that worker contains mutable state. Returning a shared instance from the supplier is a direct path to race conditions unless the worker itself is completely stateless or internally synchronized.

## Data Pipeline
ParallelTask does not operate as a traditional data pipeline component that transforms and passes data. Instead, it is a control-flow orchestrator. Its "pipeline" describes the flow of execution control.

> Flow:
> Controlling Thread (configures tasks) -> **ParallelTask.doInvoke()** -> ForkJoinPool -> Multiple Worker Threads (execute ParallelRangeTask.compute()) -> **ParallelTask** (completes) -> Controlling Thread (unblocks)


---
description: Architectural reference for ParallelRangeTask
---

# ParallelRangeTask

**Package:** com.hypixel.hytale.component.task
**Type:** Transient

## Definition
```java
// Signature
public class ParallelRangeTask<D extends IntConsumer> extends CountedCompleter<Void> {
```

## Architecture & Concepts
The ParallelRangeTask is a high-performance, specialized component for parallelizing "for-loop" style computations. It is built directly on the Java Fork/Join Framework and is designed to efficiently partition a numerical range across all available CPU cores for concurrent processing.

Its primary architectural role is to serve as a reusable abstraction over the complexities of `java.util.concurrent.CountedCompleter`. Instead of manually managing task splitting, forking, and joining, a developer can define a range and an operation, and the ParallelRangeTask orchestrates the parallel execution.

The key design features are:
- **Work Partitioning:** The `init` method divides the total numerical range into a series of smaller, non-overlapping sub-ranges. Each sub-range is assigned to a pre-allocated `SubTask`.
- **Pre-Allocation:** To minimize garbage collection pressure during computation, the task pre-allocates a fixed number of `SubTask` workers (controlled by `TASK_COUNT`). This avoids object creation overhead when the task is executed.
- **State Isolation:** The generic parameter `D extends IntConsumer` is critical. The constructor requires a `Supplier<D>`, which acts as a factory. Each `SubTask` receives its own distinct instance of `D`. This pattern is fundamental to its concurrency model, as it ensures that each worker thread operates on its own private data, eliminating the need for locks and preventing race conditions over shared state.

This component is intended for CPU-bound tasks that can be easily parallelized, such as terrain generation, image processing, or large-scale data transformation on arrays.

### Lifecycle & Ownership
- **Creation:** An instance is created directly by the developer using `new ParallelRangeTask(supplier)`. It is not managed by a dependency injection framework or service locator. The caller is the owner.
- **Scope:** The object's lifetime is typically short and bound to a single, specific computation. It is created, configured via `init`, executed by a `ForkJoinPool`, and then becomes eligible for garbage collection once the results are consumed. While it can be reused by calling `init` again, its state is tied to a single execution at a time.
- **Destruction:** There is no explicit destruction method. The Java Garbage Collector reclaims the object and its sub-tasks once they are no longer referenced.

## Internal State & Concurrency
- **State:** The ParallelRangeTask is highly stateful and mutable. It maintains the number of active sub-tasks (`size`), a `running` flag to prevent reconfiguration during execution, and an array of `SubTask` objects which themselves hold state (their assigned range and data object).
- **Thread Safety:** This class is **not thread-safe for configuration but is designed for concurrent execution**.
    - The public configuration methods (`init`, `set`) are guarded by the `running` flag. An `IllegalStateException` is thrown if they are called after computation has begun. This enforces a strict "configure-then-execute" lifecycle.
    - The `running` field is marked `volatile` to ensure that changes to its value are immediately visible to all threads, preventing stale reads.
    - The core `compute` logic is inherently concurrent and relies on the memory-fencing and scheduling guarantees of the Fork/Join framework.
    - The primary mechanism for application-level thread safety is the use of a `Supplier` to provide a unique, isolated data object to each worker thread, thereby avoiding shared mutable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(int from, int to) | ParallelRangeTask<D> | O(C) | Configures the task to process a range. Partitions the work across a constant number C of sub-tasks. Throws `IllegalStateException` if already running. |
| compute() | void | O(N/P) | Executes the task. This method is the entry point for the Fork/Join framework. N is the range size, P is the parallelism level. |
| get(int i) | D | O(1) | Retrieves the data object associated with the i-th sub-task. Used to collect results after computation. |
| set(int i, D data) | void | O(1) | Replaces the data object for a sub-task. Must be called before execution. |
| size() | int | O(1) | Returns the number of sub-tasks that will be used for the currently configured range. |

## Integration Patterns

### Standard Usage
The typical use case involves creating the task, initializing it with a range, submitting it to the common `ForkJoinPool`, and then collecting results from the individual data objects.

```java
// 1. Define a stateful consumer and a factory for it.
//    Each thread will get its own instance of MyResultCollector.
class MyResultCollector implements IntConsumer {
    public long sum = 0;
    public void accept(int value) {
        sum += value;
    }
}
Supplier<MyResultCollector> supplier = () -> new MyResultCollector();

// 2. Create and initialize the task.
ParallelRangeTask<MyResultCollector> task = new ParallelRangeTask<>(supplier);
task.init(0, 1_000_000);

// 3. Execute in the common pool and wait for completion.
ForkJoinPool.commonPool().invoke(task);

// 4. Aggregate results from each isolated data object.
long totalSum = 0;
for (int i = 0; i < task.size(); i++) {
    totalSum += task.get(i).sum;
}
```

### Anti-Patterns (Do NOT do this)
- **Shared State in Consumer:** Do not design the `Supplier` to return the same instance of a non-thread-safe `IntConsumer`. This defeats the state isolation model and will lead to severe data corruption.

    ```java
    // ANTI-PATTERN: Sharing a single, non-thread-safe instance
    MyResultCollector sharedCollector = new MyResultCollector();
    Supplier<MyResultCollector> badSupplier = () -> sharedCollector; // DANGEROUS
    ParallelRangeTask<MyResultCollector> task = new ParallelRangeTask<>(badSupplier);
    ```

- **Modification After Start:** Do not attempt to call `init` or `set` after the task has been submitted to a `ForkJoinPool`. The `running` flag will prevent this and throw an `IllegalStateException`.

- **Ignoring Results:** This task is often used for computations that produce a result. Forgetting to iterate over the sub-tasks with `get(i)` after completion means the computed work is lost.

## Data Pipeline
ParallelRangeTask is a computational primitive, not a pipeline stage. It generates and processes data in-place rather than passing it through.

> **Control Flow:**
> 1. **Configuration:** A developer provides a numerical range (`from`, `to`) and a `Supplier` for worker objects.
> 2. **Partitioning:** The `init` method divides the range across a fixed number of internal `SubTask` instances.
> 3. **Execution:** The `ForkJoinPool` invokes the main `compute` method.
> 4. **Forking:** The **ParallelRangeTask** forks `N-1` of its `SubTask` children to be executed by other worker threads. It executes the final `SubTask` on its own thread.
> 5. **Computation:** Each `SubTask` iterates over its assigned sub-range, calling `data.accept(i)` for each number.
> 6. **Completion:** As each `SubTask` finishes, it notifies the parent `ParallelRangeTask` via `propagateCompletion`. The parent task completes only after all its children have completed.
> 7. **Aggregation:** The developer's code, which was waiting for the task to complete, can now safely access the results stored in each sub-task's data object via `task.get(i)`.


---
description: Architectural reference for ForEachTaskData
---

# ForEachTaskData<ECS_TYPE>

**Package:** com.hypixel.hytale.component.data
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class ForEachTaskData<ECS_TYPE> implements IntConsumer {
```

## Architecture & Concepts
The ForEachTaskData class is a fundamental, low-level data structure within Hytale's Entity Component System (ECS) parallel processing framework. It is not a user-facing system but rather a critical *payload* object designed to encapsulate a single unit of work for execution on a worker thread.

Its primary role is to bundle the context required to process a subset of entities within an ArchetypeChunk. This context includes:
1.  **The Logic:** An IntBiObjectConsumer representing the operation to be performed on each entity.
2.  **The Data:** A reference to the ArchetypeChunk containing the raw component data.
3.  **The Output:** A thread-local CommandBuffer for recording deferred structural changes (e.g., creating or destroying entities).

By implementing the IntConsumer interface, instances of ForEachTaskData can be seamlessly integrated with parallel iteration schedulers that operate on integer indices. This design avoids passing multiple arguments to worker threads, simplifying the task scheduler's API and potentially improving performance by packaging related data together. This class is a key enabler for the massive data parallelism required by the ECS.

### Lifecycle & Ownership
The lifecycle of a ForEachTaskData instance is extremely brief and tightly controlled by the ECS task scheduling system. Developers should never manage this lifecycle manually.

-   **Creation:** Instances are expected to be managed by a high-performance object pool. The scheduler acquires a pre-allocated or new instance when building a parallel job. Direct instantiation by developers is a design violation.
-   **Scope:** The object is live only for the duration of a single parallel task invocation. It is initialized, dispatched to a worker, executed, and then cleared within one processing cycle, typically a single game tick.
-   **Destruction:** The object is not destroyed in a traditional sense. The `clear` method is invoked by the task scheduler after its results have been merged. This nullifies all internal references, effectively resetting its state and making it eligible to be returned to the object pool for immediate reuse. This pooling strategy is critical for minimizing garbage collection overhead.

## Internal State & Concurrency
-   **State:** ForEachTaskData is highly mutable by design. The `init` and `clear` methods exist solely to reconfigure its state for each new task, making it a reusable container.
-   **Thread Safety:** This class is **not thread-safe** and must be used with a strict ownership model.
    -   It is configured (via `init`) on a main or coordinator thread.
    -   It is then passed to a *single* worker thread where its `accept` method is called. It is unsafe for multiple threads to access the same instance concurrently.
    -   The `assert this.commandBuffer.setThread()` call within the `accept` method is a critical runtime check to enforce that the associated CommandBuffer is correctly bound to the executing worker thread, preventing cross-thread contamination of command lists.
    -   The static `invokeParallelTask` method provides the synchronization barrier, safely merging results from all thread-local command buffers back into a main command buffer after all parallel work is complete.

## API Surface
The public API is minimal, reflecting its role as an internal data carrier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(consumer, chunk, buffer) | void | O(1) | **Internal Use Only.** Configures the task with the logic, data chunk, and output buffer for a new job. |
| accept(index) | void | O(K) | **Internal Use Only.** Executes the consumer logic for a single entity index. K is the complexity of the consumer. |
| clear() | void | O(1) | **Internal Use Only.** Resets the object's state for object pooling. |
| invokeParallelTask(task, buffer) | static void | O(N) | Orchestrates the execution of a batch of parallel tasks and merges their results. N is the total number of sub-tasks. |

## Integration Patterns

### Standard Usage
A developer will not interact with ForEachTaskData directly. Instead, they use a high-level ECS query API, which internally builds, dispatches, and manages these data objects. The `invokeParallelTask` method is the primary entry point for the system that consumes these objects.

```java
// High-level system code that would trigger the use of ForEachTaskData
// The developer provides the lambda; the engine handles the rest.

// 1. A ParallelTask is constructed by the ECS scheduler.
ParallelTask<ForEachTaskData<MyEcsType>> task = ecs.buildParallelQuery(...);

// 2. A main CommandBuffer is prepared.
CommandBuffer<MyEcsType> mainCommandBuffer = ecs.getCommandBuffer();

// 3. The entire parallel operation is invoked.
// This static method orchestrates the lifecycle of many ForEachTaskData instances.
ForEachTaskData.invokeParallelTask(task, mainCommandBuffer);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ForEachTaskData()`. This bypasses the engine's object pooling system, which will lead to severe performance degradation due to excessive garbage collection.
-   **Manual Lifecycle Management:** Do not call `init` or `clear` from application code. These methods are strictly owned by the parallel task scheduler.
-   **Reference Leaking:** Do not store a reference to a ForEachTaskData instance outside the scope of the system that created it. The object is ephemeral and will be cleared and reused for a completely different task in a subsequent operation, making any stored reference invalid and dangerous.

## Data Pipeline
The flow of data and control through this component is a precise, multi-threaded sequence orchestrated by the ECS scheduler.

> Flow:
> 1.  ECS Scheduler acquires a pooled **ForEachTaskData** instance.
> 2.  Scheduler calls `init()` to populate it with an ArchetypeChunk, a thread-local CommandBuffer, and the user-defined system logic (consumer).
> 3.  The populated **ForEachTaskData** is added to a ParallelTask and dispatched to a worker thread.
> 4.  Worker Thread invokes `accept(index)` for each entity in its assigned range.
> 5.  The consumer logic executes, writing any structural changes (e.g., `destroyEntity`) into the thread-local CommandBuffer.
> 6.  After all workers finish, the main thread calls the static `invokeParallelTask` method.
> 7.  `invokeParallelTask` iterates through each completed **ForEachTaskData** instance.
> 8.  It merges the thread-local CommandBuffer into the main, world-level CommandBuffer.
> 9.  Finally, it calls `clear()` on the **ForEachTaskData** instance, preparing it to be returned to the object pool.


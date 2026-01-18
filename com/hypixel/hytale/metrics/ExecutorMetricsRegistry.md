---
description: Architectural reference for ExecutorMetricsRegistry
---

# ExecutorMetricsRegistry

**Package:** com.hypixel.hytale.metrics
**Type:** Specialized Registry

## Definition
```java
// Signature
public class ExecutorMetricsRegistry<T extends ExecutorMetricsRegistry.ExecutorMetric> extends MetricsRegistry<T> {
```

## Architecture & Concepts
The ExecutorMetricsRegistry is a specialized extension of the base MetricsRegistry designed to solve a critical concurrency problem: safely collecting metrics from components that are bound to a specific thread.

In a game engine, many core systems like the world simulation, physics, or entity management are not thread-safe and must only be accessed from their designated update thread. A generic metrics system attempting to scrape data from these components from a separate background thread would introduce race conditions and data corruption.

This class acts as a thread-aware proxy. It leverages the **ExecutorMetric** interface, which requires a component to both expose its own Executor (typically its update thread) and provide a mechanism to check if the current call is already on that thread via **isInThread**.

When the **encode** method is invoked, the registry performs this check.
- If the call is already on the correct thread, it behaves identically to the parent MetricsRegistry, encoding the data directly.
- If the call originates from a different thread (e.g., a global metrics-gathering thread), it transparently dispatches the encoding task to the component's own Executor. The calling thread then blocks until the task is complete and the result is returned.

This design ensures that metric collection is always thread-safe by delegating execution to the component that owns the data, upholding the thread-affinity of engine components without burdening the caller with complex synchronization logic.

## Lifecycle & Ownership
- **Creation:** Instantiated during the application bootstrap sequence by a higher-level service responsible for constructing the complete metrics pipeline, such as a central MetricsService. It is not intended for on-demand creation during the game loop.
- **Scope:** The registry's lifetime is tied to the application session. It persists as long as the server or client is running, holding the configuration for a specific set of thread-bound metrics.
- **Destruction:** De-referenced and garbage collected during application shutdown when the parent metrics services are torn down.

## Internal State & Concurrency
- **State:** The internal state is inherited from the parent MetricsRegistry, primarily consisting of a collection of registered metric providers. This state is mutable; new providers are added via the **register** methods.
- **Thread Safety:** The class is conditionally thread-safe.
    - The **encode** method is designed to be safely called from any thread. It achieves safety through thread-dispatching, not through internal locks.
    - The various **register** methods are **not thread-safe**. Registration is assumed to be a single-threaded operation performed during the initialization phase before the registry is actively used. Modifying registrations from multiple threads will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| encode(T t, ExtraInfo extraInfo) | BsonValue | Variable | Encodes the metrics from the target provider. If called off-thread, this method blocks until the task completes on the target's thread. |
| register(id, func) | ExecutorMetricsRegistry | O(1) | Registers a metric provider function. Returns self for fluent chaining. Not thread-safe. |
| register(id, func, codec) | ExecutorMetricsRegistry | O(1) | Registers a metric provider function with a custom codec. Returns self for fluent chaining. Not thread-safe. |

## Integration Patterns

### Standard Usage
This registry is used to gather metrics from a system that operates on its own thread, such as a world simulation. The metrics collector can then safely call **encode** from a separate background thread.

```java
// Assume GameWorld implements ExecutorMetric
GameWorld mainWorld = ...;

// During initialization
ExecutorMetricsRegistry<GameWorld> worldRegistry = new ExecutorMetricsRegistry<>();
worldRegistry.register("entityCount", GameWorld::getEntityCount);

// From a separate metrics-reporting thread
// The call to encode() will safely dispatch the getEntityCount() call
// to the GameWorld's update thread and block until it returns.
BsonValue metrics = worldRegistry.encode(mainWorld, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Registration:** Do not call any **register** method after the initialization phase is complete. The internal state is not protected against concurrent modification. All metrics should be registered during bootstrap.
- **Blocking Critical Threads:** The **encode** method blocks the calling thread if a thread-hop is required. Calling **encode** from a performance-critical thread (like the main render loop) to collect a metric from a *different* and potentially slow system can introduce significant stalls and frame drops. Metric collection should be performed by dedicated, low-priority background threads.
- **Incorrect ExecutorMetric Implementation:** If the **isInThread** method is implemented incorrectly, the system may either perform unnecessary and expensive thread-hops or fail to perform necessary ones, leading to race conditions.

## Data Pipeline
The data flow for an off-thread metrics collection request is a defining characteristic of this class. It ensures data is read safely from its source thread.

> Flow:
> Metrics Collector Thread -> **ExecutorMetricsRegistry.encode()** -> Check **isInThread()** -> Task dispatched via CompletableFuture -> Target Component Thread -> **super.encode()** executes -> BsonValue returned -> **join()** unblocks Collector Thread -> Final BsonValue available

---


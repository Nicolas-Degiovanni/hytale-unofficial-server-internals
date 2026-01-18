---
description: Architectural reference for SynchronizedAverageCollector
---

# SynchronizedAverageCollector

**Package:** com.hypixel.hytale.metrics.metric
**Type:** Stateful Collector / Transient

## Definition
```java
// Signature
public class SynchronizedAverageCollector extends AverageCollector {
```

## Architecture & Concepts
The SynchronizedAverageCollector is a thread-safe decorator for the base AverageCollector. Its sole purpose is to provide a concurrency control layer, enabling multiple threads to safely report data points to a single metric instance without data corruption or race conditions.

This component is critical in a multi-threaded engine environment where various subsystems (e.g., rendering, networking, physics) operate independently and need to report performance data concurrently. It employs a simple, coarse-grained locking strategy by synchronizing every method. This guarantees data integrity at the cost of potential thread contention under extremely high-frequency reporting.

It follows the Decorator pattern, extending the functionality of the parent class without altering its core logic. The parent, AverageCollector, contains the actual state and averaging algorithm, while this class acts as a synchronized proxy to that state.

### Lifecycle & Ownership
- **Creation:** Instances are typically created and managed by a central metrics registry or factory, not directly by client code. A new instance is created for each specific average metric that requires thread-safe collection.
- **Scope:** The lifetime of an instance is tied to the lifetime of the metric it represents. It persists as long as the engine is configured to track that specific metric, which is often the entire application session.
- **Destruction:** The object is eligible for garbage collection when the associated metric is unregistered or when the central metrics system is shut down. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is stateful by inheritance. It does not introduce any new state of its own but manages access to the mutable state within the parent AverageCollector, which typically includes a collection of data points, a running sum, and a count.
- **Thread Safety:** This class is **explicitly thread-safe**. It achieves this by declaring every public method as `synchronized`. This ensures that only one thread can modify or access the underlying collection at any given time, preventing inconsistent reads or concurrent modification exceptions.

**Warning:** While this approach guarantees safety, the use of a single intrinsic lock for all operations can become a performance bottleneck if multiple threads attempt to write metrics at a very high rate.

## API Surface
The public contract is inherited entirely from AverageCollector, with the addition of synchronization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | double | O(1) | Atomically retrieves the current calculated average. |
| size() | long | O(1) | Atomically retrieves the number of data points in the collection. |
| addAndGet(double v) | double | O(1) | Atomically adds a new data point and returns the new average. |
| add(double v) | void | O(1) | Atomically adds a new data point to the collection. |
| remove(double v) | void | O(N) | Atomically removes a data point. Performance is dependent on the parent implementation. |
| clear() | void | O(N) | Atomically removes all data points and resets the collector's state. |

## Integration Patterns

### Standard Usage
Client code should never instantiate this class directly. It should be retrieved from a central service, which provides the correct collector type based on the metric's requirements.

```java
// Obtain a thread-safe collector from the central metrics service
AverageCollector frameTimeCollector = Metrics.getAverage("engine.renderer.frametime");

// From the render thread, add a data point
// This call is thread-safe and will not conflict with other threads.
frameTimeCollector.add(16.67);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SynchronizedAverageCollector()`. This bypasses the metrics registration system, creating an untracked and unmanaged metric instance.
- **Using AverageCollector Directly:** In a multi-threaded context, do not use the base AverageCollector. Doing so will lead to data corruption and unpredictable behavior. This synchronized variant exists specifically to prevent that.
- **Unnecessary Synchronization:** Do not wrap calls to this class in another `synchronized` block. The object is already internally synchronized, and adding another layer of locking is redundant and may lead to deadlocks.

## Data Pipeline
This class acts as a terminal sink in a data pipeline. It collects values from various sources and makes an aggregated value available for consumption by reporting systems.

> **Data Ingestion Flow:**
>
> Producer Thread (e.g., Network Thread, Game Thread) -> Measures a value -> Calls **SynchronizedAverageCollector.add(value)** -> Internal state is safely updated.

> **Data Reporting Flow:**
>
> Metrics Exporter (e.g., JMX, Prometheus Bridge) -> Calls **SynchronizedAverageCollector.get()** -> Atomically reads the current average -> Sends to monitoring system.


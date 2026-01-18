---
description: Architectural reference for AverageCollector
---

# AverageCollector

**Package:** com.hypixel.hytale.metrics.metric
**Type:** Transient State Object

## Definition
```java
// Signature
public class AverageCollector {
```

## Architecture & Concepts
The AverageCollector is a high-performance, low-memory data structure designed to calculate a running (or moving) average of a stream of double-precision floating-point numbers. It is a fundamental component within the engine's metrics and performance monitoring systems.

Its primary architectural advantage is the use of an incremental algorithm to update the average. This avoids the need to store the entire dataset of values, resulting in a constant memory footprint, O(1), regardless of the number of samples processed. This makes it ideal for long-running, high-frequency metrics such as frame time, network latency, or entity processing time, where storing a complete history would be computationally and memory-prohibitive.

This class acts as a specialized numerical accumulator. It is not a general-purpose collection but a stateful calculator intended to be embedded within higher-level monitoring services.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its default constructor, `new AverageCollector()`. It is designed to be a member field within a component that needs to track a specific metric.
- **Scope:** The lifetime of an AverageCollector instance is strictly bound to its owning object. For example, an instance within a `NetworkSession` object would live for the duration of that network session.
- **Destruction:** The object is managed by the Java Garbage Collector. It is deallocated when its owning object is destroyed and no other references to it exist. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The AverageCollector is inherently stateful and highly mutable. Its internal state consists of two fields: `val` (the current average) and `n` (the number of samples). Nearly every method on its public API mutates this internal state.

- **Thread Safety:** **CRITICAL WARNING:** This implementation is **not thread-safe**. The read-modify-write operations on its internal fields (e.g., in the `add` method) create severe race conditions if the object is accessed concurrently from multiple threads. This will lead to silent data corruption and an inaccurate average. All concurrent access **must** be managed externally, typically by synchronizing on the owning object or using an explicit lock.

## API Surface
The public contract is designed for high-frequency updates and periodic reads.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(double v) | void | O(1) | Adds a new value to the set and updates the running average. |
| remove(double v) | void | O(1) | Reverses the effect of adding a specific value. See Anti-Patterns. |
| get() | double | O(1) | Returns the current calculated average. |
| size() | long | O(1) | Returns the total number of samples added. |
| clear() | void | O(1) | Resets the collector to its initial state (zero average, zero samples). |
| set(double v) | void | O(1) | Resets the collector and initializes it with a single value. |

## Integration Patterns

### Standard Usage
The intended pattern is to embed an AverageCollector as a private field within a manager or service that produces numerical data. The service is responsible for the collector's lifecycle and for ensuring thread-safe access if necessary.

```java
// Example: A component for tracking frame rendering times
public class FrameRateMonitor {
    // The collector is a private implementation detail of the monitor.
    private final AverageCollector frameTimeCollector = new AverageCollector();

    // This method is called once per frame from the main game loop.
    public void recordFrameTime(double deltaTimeSeconds) {
        // Because this is called from a single thread (the game loop),
        // no external synchronization is needed here.
        frameTimeCollector.add(deltaTimeSeconds * 1000.0); // Add time in ms
    }

    public double getAverageFrameTimeMs() {
        return frameTimeCollector.get();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Global Instance:** Do not expose a single AverageCollector as a static global or a shared service instance for unrelated metrics. This couples different systems and creates a high risk of thread-safety violations and logical errors where one system's `clear()` call erases data for another.

- **Misuse of remove:** The `remove` method is mathematically precise and computationally expensive. It is designed only for cases where a specific, known value must be retracted from the set. It is **not** for statistical outlier removal. Using `remove` with a value that was never added will corrupt the average.

- **Unsynchronized Concurrent Access:** Never call `add`, `remove`, or `clear` from multiple threads without external locking. This is the most common and severe error when using this class.

    ```java
    // BAD: This will corrupt the collector's state due to race conditions.
    AverageCollector sharedCollector = new AverageCollector();
    new Thread(() -> sharedCollector.add(100.0)).start();
    new Thread(() -> sharedCollector.add(200.0)).start();
    ```

## Data Pipeline
The AverageCollector serves as a terminal processing stage for a stream of raw numerical data. It transforms a sequence of individual data points into a single, aggregated metric.

> **Flow (Ingestion):**
> Raw Data Event (e.g., Frame Time, Packet Size) -> Owning Service Logic -> **AverageCollector.add(value)** -> Internal State Update

> **Flow (Reporting):**
> Metrics Polling System -> Owning Service Getter -> **AverageCollector.get()** -> Metrics Registry / Dashboard


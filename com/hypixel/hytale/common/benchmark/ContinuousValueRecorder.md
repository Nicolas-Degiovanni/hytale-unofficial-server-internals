---
description: Architectural reference for ContinuousValueRecorder
---

# ContinuousValueRecorder

**Package:** com.hypixel.hytale.common.benchmark
**Type:** Transient

## Definition
```java
// Signature
public class ContinuousValueRecorder {
```

## Architecture & Concepts
The ContinuousValueRecorder is a low-level, high-performance data structure designed for the real-time aggregation of statistical metrics. It serves as a fundamental building block within the engine's benchmarking and performance monitoring subsystems.

Its primary role is to efficiently track the minimum, maximum, count, and running average of a continuous stream of double-precision values. The design prioritizes minimal overhead per operation, making it suitable for high-frequency contexts such as per-frame or per-tick performance data collection. This class is intentionally lean, containing only the mutable state required to perform its calculations, and is not integrated with any service locators or dependency injection frameworks.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the `new` keyword. Ownership is assumed by the component that requires the measurement, such as a FrameTimeMonitor, NetworkProfiler, or a temporary benchmarking scope.
- **Scope:** The lifetime of a ContinuousValueRecorder is explicitly tied to the duration of a specific measurement task. It may be a short-lived object for a one-off benchmark or a long-lived field within a persistent monitoring service.
- **Destruction:** The object is eligible for garbage collection once its owner is destroyed and all references are released. The `reset` method is provided to enable object reuse, which is the preferred pattern for recurring measurements to avoid unnecessary heap allocations.

## Internal State & Concurrency
- **State:** This class is highly mutable. Each invocation of the `record` method modifies the internal `minValue`, `maxValue`, `sumValues`, and `count` fields. It is a stateful accumulator by design.
- **Thread Safety:** **This class is not thread-safe.** All field access and modification operations are non-atomic and unsynchronized. Concurrent calls to `record` or any other method from multiple threads will result in race conditions, data corruption, and invalid statistical results. All access must be externally synchronized or strictly confined to a single thread.

## API Surface
The public contract is designed for simplicity and performance. Getters provide access to the calculated statistics, while `record` and `reset` manage the state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| record(double value) | double | O(1) | Adds a new value to the statistical set, updating all internal aggregates. |
| reset() | void | O(1) | Resets all internal aggregates to their initial state, effectively starting a new measurement session. |
| getAverage() | double | O(1) | Returns the calculated average of all recorded values. Returns 0.0 if no values have been recorded. |
| getMinValue() | double | O(1) | Returns the minimum value recorded. Returns 0.0 if no values have been recorded. |
| getMaxValue() | double | O(1) | Returns the maximum value recorded. Returns 0.0 if no values have been recorded. |
| getCount() | long | O(1) | Returns the total number of values recorded. |

## Integration Patterns

### Standard Usage
The intended use is to instantiate the recorder, call `record` repeatedly within a loop or event handler, and then query the results. The `reset` method should be used to begin a new, distinct measurement interval without allocating a new object.

```java
// Example: Measuring task duration over a set number of iterations
ContinuousValueRecorder taskTimer = new ContinuousValueRecorder();

for (int i = 0; i < 1000; i++) {
    long start = System.nanoTime();
    executeMonitoredTask();
    long end = System.nanoTime();
    taskTimer.record((end - start) / 1_000_000.0); // Record duration in ms
}

// Report results
System.out.println("Average task time: " + taskTimer.getAverage() + " ms");

// Reuse the recorder for a different task
taskTimer.reset();
```

### Anti-Patterns (Do NOT do this)
- **Shared Instance Across Threads:** Never share a single ContinuousValueRecorder instance between multiple threads without implementing external locking. This is the most common source of error and will lead to unpredictable and incorrect data. Each thread should own its own recorder instance.
- **Forgetting to Reset:** If an instance is reused for multiple distinct measurement periods (e.g., measuring performance every 5 seconds), failing to call `reset` between periods will cause new data to be aggregated with old data, rendering the results meaningless.
- **Interpreting Zero-Count Results:** Calling getters like `getAverage` before any values have been recorded will return 0.0. Logic should always check `getCount` if the distinction between a true zero average and a no-data state is important.

## Data Pipeline
This component acts as a terminal aggregator or a data sink in a measurement pipeline. It does not forward data; it collects and summarizes it.

> Flow:
> Raw Performance Metric (e.g., Frame Time, Packet Size) -> **ContinuousValueRecorder.record()** -> Internal State Aggregation -> Reporting System (reads getAverage(), getMaxValue(), etc.)


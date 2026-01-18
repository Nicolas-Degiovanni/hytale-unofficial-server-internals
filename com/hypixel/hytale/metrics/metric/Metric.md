---
description: Architectural reference for Metric
---

# Metric

**Package:** com.hypixel.hytale.metrics.metric
**Type:** Transient

## Definition
```java
// Signature
public class Metric {
```

## Architecture & Concepts
The Metric class is a fundamental data structure within the engine's performance monitoring and telemetry framework. It is not a service or manager, but rather a stateful container for aggregating a stream of numerical data points into a statistical summary, specifically tracking the minimum, maximum, and running average.

Architecturally, it serves as a self-contained calculation and serialization primitive. The inclusion of a static `CODEC` field is a critical design choice, indicating that Metric instances are intended to be serialized and deserialized, likely for network transmission to a central metrics server or for persistence in performance logs. It functions as a specialized Data Transfer Object (DTO) that encapsulates its own aggregation logic.

This component is expected to be used in high-frequency contexts, such as tracking frame times, network packet sizes, or entity processing durations. Its design prioritizes performance of the `add` operation over other concerns.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the `new Metric()` constructor. They are typically instantiated and owned by higher-level manager classes, such as a `PerformanceProfiler` or a `NetworkMonitor`, which manage collections of different metrics.
- **Scope:** The lifecycle of a Metric object is tied to a specific measurement interval. It is short-lived and ephemeral. For example, a system measuring frames-per-second might create a Metric, populate it for one second, report the data, and then call `clear` to begin a new interval. It does not persist for the entire application session unless its owner does.
- **Destruction:** The object is eligible for garbage collection as soon as its owning manager releases its reference, typically after a measurement and reporting cycle is complete.

## Internal State & Concurrency
- **State:** The Metric class is highly mutable. Its core purpose is to accumulate state (`min`, `max`, `average`) through successive calls to the `add` method. The internal `AverageCollector` is also a mutable, stateful object.
- **Thread Safety:** **WARNING:** This class is **not thread-safe**. All state-mutating methods (`add`, `clear`, `set`, etc.) are unsynchronized. Concurrent calls to `add` from multiple threads will result in race conditions, leading to corrupt and unreliable metric data (e.g., an incorrect minimum or maximum value). Any multi-threaded access to a single Metric instance **must** be protected by external synchronization mechanisms, such as a lock or by dispatching all updates to a single, dedicated thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(long value) | void | O(1) | Adds a new data point. Updates min, max, and average. Not thread-safe. |
| clear() | void | O(1) | Resets all internal state (min, max, average) to initial values. |
| getMin() | long | O(1) | Returns the minimum value observed since the last clear. |
| getAverage() | double | O(1) | Returns the calculated average of all values since the last clear. |
| getMax() | long | O(1) | Returns the maximum value observed since the last clear. |
| set(Metric metric) | void | O(1) | Overwrites the state of this instance with the state of another. |

## Integration Patterns

### Standard Usage
A managing system should instantiate a Metric, feed it data points over a defined interval, and then read the results before clearing it for the next interval.

```java
// A manager class holds a Metric instance for a specific measurement
Metric frameTimeMetric = new Metric();

// In the main game loop or update tick
void onFrameCompleted(long duration) {
    // All access should be from a single thread or externally synchronized
    frameTimeMetric.add(duration);
}

// Periodically, e.g., once per second
void reportMetrics() {
    System.out.println("Min: " + frameTimeMetric.getMin());
    System.out.println("Avg: " + frameTimeMetric.getAverage());
    System.out.println("Max: " + frameTimeMetric.getMax());

    // Reset for the next reporting interval
    frameTimeMetric.clear();
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `add` or other mutating methods on the same Metric instance from multiple threads without an external lock. This is the most critical anti-pattern and will lead to data corruption.
- **State Accumulation:** Do not forget to call `clear` or `resetMinMax` between measurement windows. Failure to do so will cause the metric to accumulate data indefinitely, rendering the average, min, and max values meaningless for interval-based analysis.
- **Misinterpreting Initial State:** Do not read values from a newly created or cleared Metric before at least one data point has been added. The initial values are `Long.MAX_VALUE` for min and `Long.MIN_VALUE` for max, which are sentinel values, not valid measurements.

## Data Pipeline
The primary data flow for this class involves local aggregation followed by serialization for external reporting.

> Flow:
> Raw Data Source (e.g., Game Loop) -> `add(value)` -> **Metric (In-Memory Aggregation)** -> `Metric.CODEC` -> Serialized Payload (e.g., Binary) -> Network or Disk -> External Analysis System


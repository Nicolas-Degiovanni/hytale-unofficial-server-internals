---
description: Architectural reference for HistoricMetric
---

# HistoricMetric

**Package:** com.hypixel.hytale.metrics.metric
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class HistoricMetric {
```

## Architecture & Concepts

The HistoricMetric class is a specialized, high-performance data structure designed for collecting and analyzing time-series data, such as performance counters (e.g., FPS, network latency). It functions as a rolling time-window accumulator.

Its core architecture is based on a **circular buffer** (also known as a ring buffer) for storing timestamped values efficiently. This avoids costly memory re-allocations and shifting of array elements as old data expires.

A single HistoricMetric instance can maintain multiple, concurrent time windows over the same underlying data stream. For example, it can simultaneously calculate the average, minimum, and maximum values over the last 1 second, 10 seconds, and 60 seconds from a single stream of `add` calls. This is achieved by maintaining separate start pointers (`startIndices`) into the circular buffer for each defined time period.

To provide O(1) average calculations, it delegates the running average computation to an internal array of `AverageCollector` objects, one for each time period. This design ensures that querying for an average is extremely fast, regardless of the number of data points in the window. Minimum and maximum values, however, require a scan of the relevant segment of the circular buffer.

The class is intended to be configured and instantiated via its fluent `Builder`, which defines the time periods to track and the expected interval between data points. This `minimumInterval` is used to pre-calculate the required buffer size to prevent data loss within the longest time window.

## Lifecycle & Ownership

-   **Creation:** An instance is created exclusively through the `HistoricMetric.Builder`. The builder is obtained via the static factory method `HistoricMetric.builder()`. Direct instantiation is not supported. Typically, a higher-level service, such as a `MetricsService`, would own and manage HistoricMetric instances.

-   **Scope:** The lifetime of a HistoricMetric object is tied to the metric it represents. For a session-level metric like "Frames Per Second", the object would persist for the duration of the client session. For temporary, scoped metrics, it may have a shorter lifecycle.

-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit destruction method. When all references to an instance are dropped, it will be garbage collected. The `clear()` method can be used to reset its internal state for object reuse, which can be an effective pattern for reducing GC pressure.

## Internal State & Concurrency

-   **State:** HistoricMetric is a highly stateful and mutable class. Its primary purpose is to accumulate and mutate its internal state, which consists of several arrays representing the circular buffers for timestamps and values, pointers for each time window, and the state of the `AverageCollector` helpers.

-   **Thread Safety:** **This class is NOT thread-safe.** All public methods that modify or read internal state, such as `add`, `calculateMin`, `getAverage`, and `getValues`, perform non-atomic operations on shared arrays. Concurrent access from multiple threads without external synchronization will lead to race conditions, inconsistent reads, and potentially `ArrayIndexOutOfBoundsException`.

    **WARNING:** Any system using HistoricMetric must ensure that all interactions with a single instance are synchronized externally or confined to a single thread.

## API Surface

The public API is designed for data submission and statistical querying.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder(interval, unit) | static Builder | O(1) | Returns a new builder to configure and create a HistoricMetric instance. |
| add(timestamp, value) | void | O(P) | Adds a new data point. P is the number of configured periods. This is the primary write operation. |
| getAverage(periodIndex) | double | O(1) | Returns the pre-calculated running average for the specified time window. |
| calculateMin(periodIndex) | long | O(N) | Calculates the minimum value within the specified time window. N is the number of data points in the window. |
| calculateMax(periodIndex) | long | O(N) | Calculates the maximum value within the specified time window. N is the number of data points in the window. |
| clear() | void | O(B) | Resets all internal buffers and state. B is the internal buffer size. |

## Integration Patterns

### Standard Usage

The standard pattern involves using the builder to define the desired time windows, adding data points periodically, and querying the statistical methods as needed.

```java
// 1. Configure and build the metric tracker
HistoricMetric fpsMetric = HistoricMetric.builder(16, TimeUnit.MILLISECONDS)
    .addPeriod(1, TimeUnit.SECONDS)   // Track 1-second average FPS
    .addPeriod(5, TimeUnit.SECONDS)   // Track 5-second average FPS
    .build();

// 2. In the game loop, add a data point each frame
// This must be done from a single, controlled thread.
void onFrame(long frameTimeNanos, long fps) {
    fpsMetric.add(frameTimeNanos, fps);
}

// 3. In a UI or debug thread, query the data
// Access must be synchronized if this is a different thread.
void updateDebugUI() {
    double avgFpsLastSecond = fpsMetric.getAverage(0); // Index 0 corresponds to the first period added
    long maxFpsLastFiveSeconds = fpsMetric.calculateMax(1); // Index 1 for the second period
    System.out.println("Avg FPS (1s): " + avgFpsLastSecond);
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Never call `add` from one thread while calling `getAverage` or other query methods from another without an external lock. This will result in data corruption or crashes.

    ```java
    // BAD: Unsynchronized concurrent access
    new Thread(() -> {
        while(true) myMetric.add(System.nanoTime(), getValue());
    }).start();

    new Thread(() -> {
        while(true) System.out.println(myMetric.getAverage(0));
    }).start();
    ```

-   **Incorrect Period Ordering:** The builder requires that periods are added in increasing order of duration. While the builder will throw an `IllegalArgumentException`, attempting to circumvent this indicates a misunderstanding of the class's design.

-   **Ignoring `periodIndex`:** The query methods `getAverage`, `calculateMin`, and `calculateMax` all require a `periodIndex`. This index corresponds to the order in which `addPeriod` was called on the builder (0-indexed). Assuming a default or single period will lead to incorrect data retrieval.

## Data Pipeline

The HistoricMetric class is a processing and storage stage within a larger metrics collection pipeline. It does not generate data itself but transforms a stream of raw data points into aggregated statistical views.

> Flow:
> Raw Event Source (e.g., Game Loop, Network Handler) -> Data Point (timestamp, value) -> **HistoricMetric.add()** -> Internal Circular Buffers & AverageCollectors -> **HistoricMetric.getAverage()** -> Consumer (e.g., Debug UI, Analytics Service, Serializer)


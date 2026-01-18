---
description: Architectural reference for TimeRecorder
---

# TimeRecorder

**Package:** com.hypixel.hytale.common.benchmark
**Type:** Transient

## Definition
```java
// Signature
public class TimeRecorder extends ContinuousValueRecorder {
```

## Architecture & Concepts
The TimeRecorder is a high-precision, low-overhead utility for performance profiling and diagnostics. It serves as a specialized implementation of the more generic ContinuousValueRecorder, tailored specifically for measuring code execution time.

Its primary architectural role is to provide a simple and consistent interface for capturing time deltas at nanosecond resolution. It abstracts the process of interacting with the system's high-resolution timer, calculating durations, and converting these raw nanosecond values into a common unit (seconds) for statistical aggregation. The class is a foundational component within the engine's benchmarking framework, enabling developers to quickly instrument code paths and generate human-readable performance reports.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, for example `new TimeRecorder()`, by any system requiring performance measurement. It is a lightweight object designed for frequent creation and destruction.
- **Scope:** The lifetime of a TimeRecorder is intentionally brief, typically scoped to a single method or a discrete block of code being benchmarked. An instance should not be long-lived or shared across different logical benchmarks.
- **Destruction:** The object holds no native resources and requires no explicit cleanup. It is managed by the Java garbage collector and is eligible for collection as soon as it goes out of scope.

## Internal State & Concurrency
- **State:** The TimeRecorder is stateful. It inherits a mutable internal state from its parent, ContinuousValueRecorder, which tracks aggregate statistics such as count, sum, minimum, and maximum recorded values. Each call to `end` or `recordNanos` mutates this internal state.
- **Thread Safety:** This class is **not thread-safe**. All recording methods perform unsynchronized writes to internal fields. Sharing a single instance across multiple threads and calling recording methods concurrently will result in race conditions, leading to corrupted state and inaccurate metrics.

**WARNING:** Each thread performing measurements must create and use its own dedicated TimeRecorder instance. Do not store instances in shared static fields or pass them between threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | long | O(1) | Captures and returns a high-resolution timestamp from System.nanoTime. Marks the beginning of a measurement interval. |
| end(long start) | double | O(1) | Calculates the elapsed time since the provided start timestamp, records the duration internally, and returns the duration in seconds. |
| recordNanos(long nanos) | double | O(1) | Directly records a pre-calculated duration in nanoseconds. This is the core mutation method that updates all internal statistics. |
| formatValues(Formatter) | void | O(1) | Formats the aggregated statistics (Average, Min, Max, Count) into a provided Formatter instance for structured reporting. |

## Integration Patterns

### Standard Usage
The canonical use case involves pairing `start` and `end` calls around a block of code, preferably within a try-finally block to guarantee the measurement is recorded even if an exception occurs.

```java
// How a developer should normally use this
TimeRecorder timer = new TimeRecorder();
long startTime = timer.start();

try {
    // Code to be benchmarked executes here.
    // For example: complex physics simulation step.
    Thread.sleep(50); 
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} finally {
    timer.end(startTime);
}

// Output the results for analysis.
System.out.println("Benchmark Results: " + timer.toString());
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not use the same TimeRecorder instance to measure unrelated operations. This will merge the statistical data, making the results meaningless. Create a new instance for each distinct benchmark.
- **Cross-Thread Sharing:** Do not share an instance across threads. This is a severe concurrency violation that will corrupt the recorder's state.
- **Mismatched Calls:** Do not mismatch `start` and `end` calls. The value from a `start` call should only be used in the corresponding `end` call for that specific measurement.

## Data Pipeline
TimeRecorder acts as a data collector, not a pipeline processor. Its internal flow transforms raw system time into aggregated statistical data.

> Flow:
> System Clock (`System.nanoTime`) -> **TimeRecorder.start()** -> [Monitored Code Execution] -> System Clock (`System.nanoTime`) -> **TimeRecorder.end()** -> Internal State Mutation (Count, Sum, Min, Max) -> **TimeRecorder.toString()** or **formatValues()** -> Formatted String Output


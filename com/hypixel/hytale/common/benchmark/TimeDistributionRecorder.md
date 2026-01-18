---
description: Architectural reference for TimeDistributionRecorder
---

# TimeDistributionRecorder

**Package:** com.hypixel.hytale.common.benchmark
**Type:** Transient

## Definition
```java
// Signature
public class TimeDistributionRecorder extends TimeRecorder {
```

## Architecture & Concepts
The TimeDistributionRecorder is a specialized performance analysis tool designed to capture not just aggregate timing statistics, but the *distribution* of those timings. It extends the base TimeRecorder, which tracks metrics like count, total time, and average time.

The core concept of this class is the use of a **logarithmic histogram**. Performance measurements in software engineering often span several orders of magnitude, from microseconds to hundreds of milliseconds. A linear histogram would be impractical, requiring an enormous number of bins to maintain precision for small values or suffering from poor resolution for large values.

This recorder solves the problem by mapping time values to bins on a logarithmic (base 10) scale. This provides high resolution for frequent, low-duration events (e.g., 1-10 microseconds) and progressively lower resolution for infrequent, high-duration events (e.g., 10-100 milliseconds). This is ideal for identifying performance outliers and understanding the typical operational cost of a system.

It functions as a data collector, intended to be embedded within a system under test to measure a specific, recurring operation.

### Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, typically by a higher-level benchmark or profiling manager. The creator must provide sensible time boundaries (minSecs, maxSecs) that reflect the expected range of measurements.
- **Scope:** The object's lifetime is tied to a specific profiling session or benchmark run. It is not a global or long-lived service. It is created, used to accumulate data, and then discarded after its data has been reported.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and has no explicit `destroy` or `close` method. It becomes eligible for collection as soon as its owning manager releases its reference.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its primary internal state is the `valueBins` long array, which stores the histogram counts. Each call to `recordNanos` modifies this array and the aggregate statistics inherited from the parent TimeRecorder.
- **Thread Safety:** **This class is not thread-safe.** The `recordNanos` method performs a non-atomic read-modify-write operation on the `valueBins` array (`valueBins[...]++`). Concurrent calls from multiple threads will result in lost writes and data corruption. If this recorder must be used in a multi-threaded context, all access, particularly to `recordNanos` and `reset`, must be protected by external synchronization mechanisms (e.g., a `synchronized` block or a `Lock`).

## API Surface
The public API is focused on data recording and result formatting.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| recordNanos(long nanos) | double | O(1) | Records a single nanosecond measurement. Converts it to a logarithmic index and increments the corresponding bin. |
| reset() | void | O(N) | Resets all inherited statistics and zeroes out all histogram bins. N is the number of bins. |
| timeToIndex(double secs) | int | O(1) | Converts a time in seconds to its corresponding logarithmic bin index. |
| indexToTime(int index) | double | O(1) | Converts a bin index back to the representative time value for that bin's upper bound. |
| get(int index) | long | O(1) | Returns the raw count for the specified bin index. |
| size() | int | O(1) | Returns the total number of bins in the histogram. |

## Integration Patterns

### Standard Usage
The recorder is instantiated, used within a loop to measure a repeated operation, and then its results are extracted by a reporting system.

```java
// 1. A profiling manager creates a recorder for a specific system.
//    The range is set from 10ms down to 1 microsecond.
TimeDistributionRecorder physicsStepRecorder = new TimeDistributionRecorder(0.01, 1.0E-6);

// 2. Inside the main game loop, the operation is timed.
for (int i = 0; i < 1000; i++) {
    long startTime = System.nanoTime();
    physicsEngine.step();
    long duration = System.nanoTime() - startTime;

    // 3. The measurement is recorded.
    physicsStepRecorder.recordNanos(duration);
}

// 4. After the run, a reporting system formats and prints the results.
//    This typically uses the formatValues or toString methods.
System.out.println(physicsStepRecorder.toString());
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Recording:** Do not call `recordNanos` from multiple threads without an external lock. This will lead to race conditions and inaccurate counts.
- **Mismatched Range:** Initializing the recorder with a `maxSecs` and `minSecs` range that does not align with the actual measurements will cause all data to cluster in the first or last bin, rendering the distribution data useless.
- **Reuse Without Reset:** Using the same recorder instance for multiple distinct benchmark runs without calling `reset()` between runs will merge the results, corrupting the data for all subsequent runs.

## Data Pipeline
The flow of a single measurement through the recorder is linear and synchronous.

> Flow:
> Raw Nanosecond Value (long) -> **TimeDistributionRecorder.recordNanos** -> Logarithmic Index Calculation -> `valueBins[index]++` -> Formatter -> Textual Report


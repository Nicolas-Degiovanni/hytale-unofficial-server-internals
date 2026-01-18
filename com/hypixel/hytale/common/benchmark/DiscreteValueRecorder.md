---
description: Architectural reference for DiscreteValueRecorder
---

# DiscreteValueRecorder

**Package:** com.hypixel.hytale.common.benchmark
**Type:** Transient

## Definition
```java
// Signature
public class DiscreteValueRecorder {
```

## Architecture & Concepts
The DiscreteValueRecorder is a low-level, stateful utility for aggregating simple statistical data about a set of discrete long integer values. Its primary role within the engine is to serve as a lightweight and high-performance accumulator for performance benchmarking and profiling tasks.

It is designed to have minimal overhead during the recording phase, deferring calculations like the average to the read phase. This makes it suitable for use inside performance-critical loops where the cost of measurement must be as low as possible. It captures four fundamental metrics: the minimum value, maximum value, total count, and the arithmetic mean (average).

This class is a foundational component of the benchmarking system, providing the raw data collection mechanism upon which more complex reporting and analysis tools are built.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new DiscreteValueRecorder()`. It is not managed by a dependency injection container or a central registry.
- **Scope:** The lifecycle of a DiscreteValueRecorder is manually managed and is typically scoped to a single, specific measurement task, such as one run of a benchmark. It is intended to be short-lived.
- **Destruction:** The object is eligible for garbage collection as soon as it falls out of scope. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The DiscreteValueRecorder is highly mutable. Its internal state, consisting of the minimum, maximum, sum, and count, is modified with every call to the `record` method. The state can be completely erased and reset to its initial condition by calling `reset`.

- **Thread Safety:** **This class is not thread-safe.** Its methods are not synchronized. Concurrent calls to the `record` method from multiple threads will result in race conditions, leading to data corruption and inaccurate statistical results.

    **WARNING:** If a DiscreteValueRecorder instance must be shared across threads, all access to it, particularly write operations via `record`, must be protected by external synchronization mechanisms such as locks or atomic operations on a shared instance. Failure to do so will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| record(long value) | void | O(1) | Adds a new value to the dataset, updating internal statistics. This is the primary write operation. |
| reset() | void | O(1) | Resets all internal statistics to their initial state, preparing the instance for a new measurement session. |
| getAverage() | long | O(1) | Calculates and returns the average of all recorded values. Returns 0 if no values have been recorded. |
| getMinValue() | long | O(1) | Returns the minimum value recorded. Returns 0 if no values have been recorded. |
| getMaxValue() | long | O(1) | Returns the maximum value recorded. Returns 0 if no values have been recorded. |
| getCount() | long | O(1) | Returns the total number of values recorded. |
| formatValues(Formatter) | void | O(1) | Formats the calculated statistics into a provided Formatter instance for reporting. |

## Integration Patterns

### Standard Usage
The canonical use case is to instantiate the recorder, perform an operation within a loop, record a measurement for each iteration, and finally report the results.

```java
// Example: Benchmarking a hypothetical 'processChunk' method
DiscreteValueRecorder recorder = new DiscreteValueRecorder();
int iterations = 1000;

for (int i = 0; i < iterations; i++) {
    long startTime = System.nanoTime();
    processChunk(chunks[i]);
    long endTime = System.nanoTime();
    
    // Record the duration in microseconds
    recorder.record((endTime - startTime) / 1000);
}

System.out.println("Chunk Processing Stats (us): " + recorder.toString());
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share a single instance across multiple threads for writing without external locking. This is the most critical anti-pattern and will silently corrupt your data.

    ```java
    // DO NOT DO THIS
    DiscreteValueRecorder sharedRecorder = new DiscreteValueRecorder();
    // Thread 1
    new Thread(() -> sharedRecorder.record(100)).start();
    // Thread 2
    new Thread(() -> sharedRecorder.record(200)).start();
    // The final state of sharedRecorder is now unpredictable.
    ```

- **Instance Reuse Without Reset:** If an instance is reused for multiple distinct benchmark runs, failing to call `reset` between runs will contaminate the new results with data from the previous run.

## Data Pipeline
The data flow for this component is simple and self-contained. It acts as a terminal collector of raw numerical data for the purpose of aggregation.

> Flow:
> Raw Measurement (e.g., nanoseconds) -> **DiscreteValueRecorder.record()** -> Internal State Update (min, max, sum, count) -> **get...() / format...()** -> Formatted Report / Log Output


---
description: Architectural reference for TimeInstrument
---

# TimeInstrument

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.performanceinstruments
**Type:** Transient

## Definition
```java
// Signature
public class TimeInstrument {
    // ...
    public static class Probe {
        // ...
    }
}
```

## Architecture & Concepts
TimeInstrument is a high-precision performance profiling utility designed to measure and aggregate the execution time of repetitive, structured operations. It is not a general-purpose profiler but a specialized tool for analyzing consistent code paths, such as world generation stages, entity updates, or rendering frames.

The core architectural concept is a tree of **Probe** objects. A single Probe measures a discrete block of code, while its children measure sub-blocks, forming a hierarchy. The TimeInstrument class acts as an aggregator, collecting multiple complete Probe trees—referred to as "samples"—and averaging their results.

A critical design constraint is **structural compatibility**. The first sample provided to a TimeInstrument instance establishes a structural baseline. All subsequent samples must have an identical Probe tree structure. If a sample with a mismatched structure is provided (e.g., due to a conditional code path not taken in the first sample), it is discarded with a warning. This ensures that the aggregated data is a true average of the same logical operation performed multiple times.

## Lifecycle & Ownership
- **Creation:** Manually instantiated by client code that needs to perform a performance measurement. It is not managed by a service locator or dependency injection framework.
- **Scope:** The lifetime of a TimeInstrument is typically bound to the scope of the measurement task. For example, an instance might be created before a loop, used to collect samples within the loop, and then discarded after its final report is generated via the toString method.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is safe to let the instance fall out of scope once the performance report is no longer needed.

## Internal State & Concurrency
- **State:** TimeInstrument is a highly stateful, mutable object. It maintains an internal sample count and an aggregated totalProbe, which is a cumulative representation of all valid samples taken. The nested Probe class is also stateful, transitioning through NOT_STARTED, STARTED, and COMPLETED states.

- **Thread Safety:** **This class is not thread-safe.** All methods that modify internal state, including takeSample and the start/stop methods on its Probe objects, are unsynchronized. Concurrent access to a single TimeInstrument instance from multiple threads will lead to race conditions, inconsistent state, and unpredictable behavior.

    **WARNING:** Each thread performing measurements must own its own distinct TimeInstrument instance. Do not share instances across threads.

## API Surface
The primary interaction is with the TimeInstrument itself and its nested Probe class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TimeInstrument(header) | constructor | O(1) | Creates a new instrument for collecting performance samples. |
| takeSample(probe) | void | O(N) | Adds a completed Probe tree as a sample. N is the number of nodes in the probe tree. Throws assertion errors if probe structures are incompatible. |
| toString() | String | O(N) | Generates a formatted, human-readable performance report based on all collected samples. N is the number of nodes in the aggregated probe tree. |
| Probe.start() | Probe | O(1) | Starts the nanosecond timer for this measurement block. |
| Probe.stop() | Probe | O(1) | Stops the timer and calculates the total elapsed time. Must be called after start. |
| Probe.createProbe(name) | Probe | O(1) | Creates and registers a child probe to measure a sub-task. |

## Integration Patterns

### Standard Usage
The intended pattern involves creating a new Probe tree for each distinct operation being measured. This tree is then passed as a single sample to the TimeInstrument.

```java
// How a developer should normally use this
TimeInstrument instrument = new TimeInstrument("Chunk Generation Performance");

for (int i = 0; i < 100; i++) {
    // A new root probe must be created for each sample.
    TimeInstrument.Probe rootProbe = new TimeInstrument.Probe("Total Generation");
    rootProbe.start();

    // Profile a sub-task
    TimeInstrument.Probe noiseProbe = rootProbe.createProbe("Noise Calculation");
    noiseProbe.start();
    // ... perform noise calculation ...
    noiseProbe.stop();

    // Profile another sub-task
    TimeInstrument.Probe structureProbe = rootProbe.createProbe("Structure Placement");
    structureProbe.start();
    // ... place structures ...
    structureProbe.stop();

    rootProbe.stop();

    // Submit the entire sample
    instrument.takeSample(rootProbe);
}

// Log the final averaged report
LoggerUtil.getLogger().info(instrument.toString());
```

### Anti-Patterns (Do NOT do this)
- **Shared Instances:** Do not share a single TimeInstrument instance across multiple threads. This will corrupt the internal state. Create one instance per thread.
- **Dynamic Probe Structures:** Do not attempt to use this tool on code paths where the sequence or existence of profiled sub-tasks changes between executions. This will result in discarded samples and inaccurate reports.
- **Reusing Probes:** Do not reuse a Probe object across multiple samples. A new Probe tree must be constructed from scratch for every call to takeSample.
- **Incomplete Probes:** Failure to call stop on a Probe that has been started will result in an assertion error when its parent is stopped or its total time is accessed.

## Data Pipeline
TimeInstrument acts as a terminal aggregator for performance data. It does not forward data to other systems; it transforms raw timing data into a formatted string report.

> Flow:
> System.nanoTime() -> **Probe.start/stop** -> Completed Probe Tree -> **TimeInstrument.takeSample** -> Internal State Aggregation -> **TimeInstrument.toString()** -> Log Output


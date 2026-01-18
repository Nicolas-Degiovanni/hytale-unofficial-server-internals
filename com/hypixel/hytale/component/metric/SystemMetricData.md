---
description: Architectural reference for SystemMetricData
---

# SystemMetricData

**Package:** com.hypixel.hytale.component.metric
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SystemMetricData {
```

## Architecture & Concepts
SystemMetricData is a passive data structure, or Data Transfer Object, designed to encapsulate a snapshot of performance metrics for a single game system at a specific point in time. It serves as a standardized container for performance data that can be serialized, transmitted over the network, or written to disk for debugging and analysis.

The defining architectural feature is the static final **CODEC** field. This implementation of the self-describing data pattern makes the class responsible for its own serialization and deserialization logic. The engine's core serialization services rely on this public contract to encode and decode instances of SystemMetricData without needing to know its internal structure. This decouples the metrics collection pipeline from the specific data being collected, allowing new metrics to be added without modifying the core pipeline.

This class aggregates multiple forms of performance data:
*   **Counters:** Simple integer values like entityCount and archetypeChunkCount.
*   **Time-Series Data:** The nullable HistoricMetric field for tracking values over time.
*   **Aggregated Results:** The MetricResults field for more complex, structured performance measurements.

## Lifecycle & Ownership
- **Creation:** Instances are created under two conditions:
    1.  By a game system (e.g., PhysicsSystem, AISystem) during its update tick to capture its current performance state.
    2.  By the engine's serialization framework, which uses the public no-argument constructor and the static CODEC to reconstruct an object from a byte stream.
- **Scope:** SystemMetricData objects are **transient and short-lived**. Their scope is confined to a single metrics-reporting transaction. They are not designed to be held in memory for extended periods.
- **Destruction:** The object becomes eligible for garbage collection as soon as it has been processed by the metrics pipeline (e.g., after being serialized to a network buffer). There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is **mutable** upon creation. The object is instantiated, its fields are populated, and then it is passed to another system. After this handoff, it should be treated as effectively immutable. Modifying its state after submission to the metrics pipeline is a severe anti-pattern.
- **Thread Safety:** This class is **not thread-safe**. It contains no locks or other concurrency primitives. It is designed to be created, populated, and read within a single thread, typically the main game thread or a dedicated system thread.

**WARNING:** Sharing an instance of SystemMetricData across threads without external synchronization will result in data corruption and undefined behavior.

## API Surface
The public API is intentionally minimal, consisting only of constructors. The primary interaction mechanism is not direct method calls, but serialization and deserialization via the static CODEC field.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SystemMetricData() | Constructor | O(1) | Creates an empty instance. Primarily for use by the serialization framework. |
| SystemMetricData(name, ...) | Constructor | O(1) | Creates a fully-populated instance. This is the standard constructor for programmatic creation. |

## Integration Patterns

### Standard Usage
The intended use is for a game system to create, populate, and immediately submit the object to a central metrics service. The creating system does not retain a reference.

```java
// A game system captures its metrics and reports them
MetricResults currentMetrics = capturePerformanceData();
HistoricMetric timingData = getHistoricTimings();

SystemMetricData report = new SystemMetricData(
    "PhysicsSystem",
    getChunkCount(),
    getEntityCount(),
    timingData,
    currentMetrics
);

// Submit to the central service for processing
metricsManager.submit(report);
```

### Anti-Patterns (Do NOT do this)
- **State Modification After Handoff:** Never modify a SystemMetricData object after passing it to another service. This violates the "snapshot" principle and can lead to inconsistent or corrupt metrics.
- **Object Re-use:** Do not attempt to pool or re-use SystemMetricData instances. They are lightweight objects, and the overhead of managing a pool outweighs the cost of garbage collecting these transient objects.
- **Long-Term Storage:** Do not hold references to these objects in long-lived collections. They are meant to represent a point-in-time snapshot, not a persistent state.

## Data Pipeline
SystemMetricData acts as a payload within the engine's performance monitoring pipeline. Its flow is unidirectional and linear.

> Flow:
> Game System Tick -> **SystemMetricData (Instantiation)** -> MetricsManager Service -> Serializer (using SystemMetricData.CODEC) -> Network Buffer or Log File


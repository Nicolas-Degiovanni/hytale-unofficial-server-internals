---
description: Architectural reference for JVMMetrics
---

# JVMMetrics

**Package:** com.hypixel.hytale.metrics
**Type:** Utility

## Definition
```java
// Signature
public class JVMMetrics {
```

## Architecture & Concepts

The JVMMetrics class is a static schema definition and data provider for comprehensive Java Virtual Machine telemetry. It does not perform any active monitoring itself; rather, it declaratively defines *what* metrics to collect and *how* to structure and serialize them.

This class serves as the primary bridge between the standard Java Management Extensions (JMX) framework, exposed via the `java.lang.management` MXBeans, and Hytale's internal metrics serialization system, which is built upon the `MetricsRegistry` and `Codec` abstractions.

The core architectural pattern is the use of static `MetricsRegistry` instances to build a hierarchical and reusable schema. The main `METRICS_REGISTRY` acts as the root of a tree of metric definitions. It composes smaller, specialized registries (e.g., `MEMORY_POOL_METRICS_REGISTRY`, `GARBAGE_COLLECTOR_METRICS_REGISTRY`) to create a deeply nested and structured data model for the entire JVM state. All definitions are configured once within a static initializer block, ensuring a consistent and immutable schema for the application's lifetime.

## Lifecycle & Ownership

-   **Creation:** JVMMetrics is never instantiated. Its static fields, primarily the `MetricsRegistry` instances, are initialized by the JVM class loader when the class is first referenced. This process is guaranteed to happen only once during the application's startup sequence.
-   **Scope:** The defined metric schemas are application-scoped and persist for the entire lifetime of the JVM process.
-   **Destruction:** All associated resources are reclaimed only when the JVM process terminates. There is no manual cleanup mechanism.

## Internal State & Concurrency

-   **State:** The class itself is stateless. Its public fields are `MetricsRegistry` instances which contain immutable definitions for data collection. These definitions consist of lambda functions and `Codec` instances, which are configured once during class loading and are not modified at runtime.
-   **Thread Safety:** This class is inherently thread-safe. The registries are populated within a static initializer, which is executed by a single thread during class loading before any concurrent access is possible. Subsequent reads of the registries are safe. The data-fetching lambdas primarily call methods on the `java.lang.management` MXBeans, which are themselves specified to be thread-safe.

**WARNING:** While the class is thread-safe, some of the data collection operations it defines can be extremely expensive. For example, the "Threads" metric performs a full stack dump of all live threads, which can introduce significant latency and should not be polled at a high frequency.

## API Surface

The public API consists exclusively of static `MetricsRegistry` fields. These registries serve as pre-configured schemas for use by telemetry and debugging systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| METRICS_REGISTRY | MetricsRegistry<Void> | N/A | The primary, top-level registry containing the comprehensive schema for all JVM metrics. This is the main entry point for a full JVM snapshot. |
| CLASS_LOADER_METRICS_REGISTRY | MetricsRegistry<ClassLoader> | N/A | A specialized, reusable schema for serializing ClassLoader information. |
| MEMORY_USAGE_METRICS_REGISTRY | MetricsRegistry<MemoryUsage> | N/A | A specialized, reusable schema for serializing MemoryUsage objects from the JMX framework. |
| GARBAGE_COLLECTOR_METRICS_REGISTRY | MetricsRegistry<GarbageCollectorMXBean> | N/A | A specialized, reusable schema for serializing detailed garbage collector statistics. |
| MEMORY_POOL_METRICS_REGISTRY | MetricsRegistry<MemoryPoolMXBean> | N/A | A specialized, reusable schema for serializing statistics about individual memory pools (e.g., Eden Space, Old Gen). |

## Integration Patterns

### Standard Usage

A telemetry service or diagnostics engine would use the static `METRICS_REGISTRY` as a schema to serialize a complete snapshot of the current JVM state. The registry is applied to a null input because the data sources are global singletons accessed via `ManagementFactory`.

```java
// In a hypothetical TelemetryService:

// 1. Obtain the master JVM metrics schema defined by JVMMetrics.
MetricsRegistry<Void> jvmSchema = JVMMetrics.METRICS_REGISTRY;

// 2. Use a serialization engine to encode the live JVM state using the schema.
// The schema is applied to a null input, as its internal data fetchers
// access global state (e.g., ManagementFactory.getThreadMXBean()).
ByteBuffer buffer = ByteBuffer.allocate(2 * 1024 * 1024); // 2MB
SerializationEngine.encode(buffer, jvmSchema, null);

// 3. The buffer now contains the complete, structured, and serialized
// JVM metrics snapshot, ready for transmission or analysis.
// telemetryBackend.send(buffer);
```

### Anti-Patterns (Do NOT do this)

-   **Schema Modification:** Do not attempt to call `register` or otherwise modify the public `MetricsRegistry` instances at runtime. They are configured once at startup and are considered immutable for the application's lifetime. Modifying them can lead to unpredictable behavior in the telemetry system.
-   **High-Frequency Polling:** The "Threads" metric definition invokes `ThreadMXBean.dumpAllThreads(true, true)`, which triggers a JVM safepoint and is a heavyweight operation. Building a metrics snapshot from `METRICS_REGISTRY` should be done infrequently (e.g., every 30-60 seconds) to avoid impacting application performance.

## Data Pipeline

JVMMetrics acts as a data source, not a processing stage. It defines the mechanism for pulling data from the JVM and structuring it for consumption by other systems.

> Flow:
> JVM Internal State -> JMX Beans (e.g., ThreadMXBean, MemoryMXBean) -> **JVMMetrics Data-Fetching Lambdas** -> MetricsRegistry Schema Application -> Serialized Output (e.g., ByteBuffer)


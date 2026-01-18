---
description: Architectural reference for MetricProvider
---

# MetricProvider

**Package:** com.hypixel.hytale.metrics
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface MetricProvider {
```

## Architecture & Concepts
The MetricProvider interface is a fundamental contract within the engine's observability and performance monitoring framework. It establishes a standardized "probe" or "sensor" pattern, allowing any system component to expose its internal state for collection and analysis.

This interface decouples the metrics aggregation system from the specific components being measured. Systems like networking, rendering, entity management, or physics can implement MetricProvider to provide performance counters, gauges, and timers. A central service, such as a MetricsManager, can then discover and poll these providers without any direct knowledge of their internal implementation. This promotes modularity and allows new systems to be instrumented by simply implementing this contract.

The static helper method, *maybe*, facilitates integration with modern functional programming patterns, particularly when processing collections of heterogeneous objects where only some may be metric providers.

## Lifecycle & Ownership
As an interface, MetricProvider itself has no lifecycle. Its lifecycle is intrinsically tied to the object that implements it.

- **Creation:** The interface is not instantiated. A concrete class (e.g., NetworkManager, WorldRenderer) implements the interface. The creation of that concrete class is handled by its respective owner, such as a service locator or dependency injection framework during application bootstrap.
- **Scope:** The ability to provide metrics exists for the entire lifetime of the implementing object. If a World object implements MetricProvider, its metrics are available as long as that World is loaded.
- **Destruction:** The contract is fulfilled until the implementing object is de-referenced and subsequently garbage collected. No explicit cleanup is required by the interface itself.

## Internal State & Concurrency
- **State:** The interface is stateless. However, any non-trivial implementation will be stateful, as its purpose is to report on the state of an underlying system. The toMetricResults method is designed as a read-only snapshot operation on that state.

- **Thread Safety:** **CRITICAL:** The interface itself makes no guarantees about thread safety. It is the absolute responsibility of the *implementing class* to ensure that the toMetricResults method is thread-safe. The central metrics aggregator may poll providers from a dedicated background thread. Implementations must use appropriate concurrency controls (e.g., locks, atomic variables, concurrent collections, or copy-on-write data structures) to prevent race conditions, inconsistent reads, or deadlocks. Failure to do so can lead to severe application instability.

## API Surface
The public contract is minimal, focusing on a single point of data extraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toMetricResults() | MetricResults | Implementation-Dependent | Queries the component for a snapshot of its current metrics. May return null if no metrics are available at the time of the call. |
| maybe(func) | Function | O(1) | A static factory method that wraps a given function. It produces a new function that safely attempts to cast the result to a MetricProvider, returning null on failure. |

## Integration Patterns

### Standard Usage
A central metrics service typically maintains a collection of registered MetricProvider instances. It polls them on a regular interval to aggregate data for reporting.

```java
// A central aggregator polls a registered provider
MetricProvider networkMetrics = serviceRegistry.get(NetworkManager.class);

if (networkMetrics != null) {
    MetricResults results = networkMetrics.toMetricResults();
    if (results != null) {
        telemetryService.submit(results);
    }
}
```

The *maybe* helper is useful for processing streams of objects that might not all be providers.

```java
// Using the 'maybe' helper with a stream
List<Object> gameSystems = List.of(new NetworkManager(), new SoundSystem());

List<MetricResults> allResults = gameSystems.stream()
    .map(MetricProvider.maybe(system -> system))
    .filter(Objects::nonNull)
    .map(MetricProvider::toMetricResults)
    .filter(Objects::nonNull)
    .collect(Collectors.toList());
```

### Anti-Patterns (Do NOT do this)
- **Blocking Implementations:** Never perform blocking I/O, complex calculations, or acquire long-held locks within the toMetricResults method. This method may be called on a frequent, time-sensitive tick, and blocking will degrade overall application performance.
- **Unsafe Concurrency:** Do not implement toMetricResults without considering that it may be called from a different thread than the one modifying the underlying state. This is a common source of data races and crashes.
- **Ignoring Nulls:** The contract explicitly allows toMetricResults to return null. Consumers of the interface must be written defensively to handle this case without throwing a NullPointerException.

## Data Pipeline
MetricProvider acts as a primary data source in the engine's telemetry pipeline.

> Flow:
> Internal State of a System (e.g., NetworkManager) -> `toMetricResults()` call -> **MetricResults** object -> Metrics Aggregator Service -> Telemetry Backend / Developer Console


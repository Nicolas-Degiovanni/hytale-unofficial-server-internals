---
description: Architectural reference for MetricSystem
---

# MetricSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface MetricSystem<ECS_TYPE> {
```

## Architecture & Concepts
The MetricSystem interface defines a standardized contract for collecting and reporting performance and state metrics from an Entity Component System (ECS) data store. It serves as a crucial component of the engine's observability and diagnostics framework.

By abstracting the process of metric collection, this interface decouples the core game systems from the monitoring and telemetry infrastructure. Any system that manages a Store of component data can have a corresponding MetricSystem implementation to expose its internal state for debugging, performance tuning, or live monitoring.

The use of a generic parameter, ECS_TYPE, ensures that this contract is adaptable and can be implemented for any type of component data, promoting a consistent approach to metric gathering across the entire engine.

## Lifecycle & Ownership
As an interface, MetricSystem itself does not have a lifecycle. The lifecycle described here pertains to concrete classes that **implement** this interface.

- **Creation:** Implementations are typically instantiated and registered by a central service, such as a SystemManager or a dedicated MetricRegistry, during the engine's bootstrap sequence or upon loading a specific world or server instance.
- **Scope:** The lifetime of a MetricSystem implementation is tightly coupled to the lifetime of the system or Store it is designed to monitor. It persists as long as the target data source is active.
- **Destruction:** An implementation is destroyed and de-registered when its corresponding system is shut down or the game session ends.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, concrete implementations may be stateful. For example, an implementation might accumulate data over several frames to calculate averages or deltas. The primary operation, toMetricResults, is designed to act on a snapshot of data passed into it, which is a common pattern for isolating state.
- **Thread Safety:** This interface is not inherently thread-safe; safety is the responsibility of the implementing class. Implementations must be designed with concurrency in mind, as metric collection is often performed on a separate, low-priority thread to avoid impacting the main game loop.

**WARNING:** Implementations of MetricSystem should treat the input Store as read-only to prevent race conditions and state corruption. Any modifications to the Store must be performed by the owning system on the appropriate thread.

## API Surface
The public contract consists of a single method for transforming raw component data into a structured metrics report.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toMetricResults(Store) | MetricResults | Implementation-dependent | Transforms the data within the provided Store into a standardized MetricResults object. Complexity can range from O(1) to O(N) based on the number of entities in the store. |

## Integration Patterns

### Standard Usage
A central monitoring service retrieves a registered MetricSystem implementation and periodically invokes it, typically on a background thread, to gather data without blocking the main game loop.

```java
// A central monitor polls a registered metric system
MetricSystem<PhysicsComponent> physicsMetrics = metricRegistry.getSystem(PhysicsComponent.class);
Store<PhysicsComponent> physicsStore = world.getStore(PhysicsComponent.class);

// This call is often scheduled on a separate thread
MetricResults results = physicsMetrics.toMetricResults(physicsStore);
telemetryService.submit(results);
```

### Anti-Patterns (Do NOT do this)
- **Frequent Polling on Game Thread:** Do not call toMetricResults repeatedly on the main game loop. Expensive implementations can introduce significant frame rate stutter. Offload metric collection to a dedicated diagnostics thread.
- **State Modification:** An implementation of toMetricResults must not modify the state of the Store passed to it. This would violate separation of concerns and introduce unpredictable behavior and concurrency bugs.

## Data Pipeline
This interface acts as a transformation point, converting live, in-memory game state into a serializable, structured format suitable for analysis.

> Flow:
> Live ECS Store -> **MetricSystem Implementation** -> MetricResults Object -> Telemetry Service / Debug UI / Logger


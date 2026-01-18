---
description: Architectural reference for EnvironmentProvider
---

# EnvironmentProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.environmentproviders
**Type:** Strategy Interface / Abstract Base Class

## Definition
```java
// Signature
public abstract class EnvironmentProvider {
```

## Architecture & Concepts
The EnvironmentProvider is an abstract base class that defines a fundamental contract within the procedural world generation system. It represents a queryable, three-dimensional data source, often referred to as an "environment". This abstraction allows different generation algorithms to sample environmental values—such as temperature, humidity, density, or raw noise fields—at any given world coordinate without being coupled to the specific implementation of how that data is generated.

This class is a classic example of the **Strategy Pattern**. The world generator is configured with concrete implementations of EnvironmentProvider, each responsible for a specific environmental variable. This decouples the high-level generation logic from the low-level noise functions or data lookups.

The inclusion of a `WorkerIndexer.Id` within the required `Context` object is a critical design choice. It signals that this system is built for a high-throughput, multi-threaded environment. Implementations are expected to be called concurrently from a pool of world generation workers, and their design must account for this.

## Lifecycle & Ownership
-   **Creation:** Concrete instances of EnvironmentProvider are not created directly by worker threads. They are instantiated and configured centrally by the world generator's orchestrator or configuration loader at the start of a generation session. This ensures all workers operate on the same consistent set of environmental parameters (e.g., world seed).
-   **Scope:** An EnvironmentProvider instance persists for the duration of a single world generation task. It is effectively a session-scoped object, shared by reference among all worker threads participating in that task.
-   **Destruction:** The object is eligible for garbage collection once the world generation session concludes and all references from the generator orchestrator and worker threads are released. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The base class is stateless. However, concrete subclasses are expected to be stateful, holding configuration such as noise generator instances, world seeds, frequency/amplitude parameters, or lookup tables. **Warning:** To ensure deterministic and reproducible world generation, this state must be considered immutable after its initial configuration.
-   **Thread Safety:** **This is a critical concern.** The `getValue` method is guaranteed to be invoked from multiple threads simultaneously. All implementations of EnvironmentProvider **must be unconditionally thread-safe**. The safest and most performant approach is to ensure the provider is either completely stateless or its state is immutable. Any use of mutable shared state will introduce severe race conditions, leading to non-deterministic world generation and visual artifacts like chunk borders.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(Context) | int | Varies | **Abstract.** The core contract. Returns the environmental value at the given position. Complexity depends entirely on the subclass implementation (e.g., O(1) for a constant, O(log N) or higher for complex noise). |
| noEnvironmentProvider() | EnvironmentProvider | O(1) | Static factory that returns a default, zero-value provider. This implements the Null Object Pattern, preventing null pointer exceptions in generator logic. |

## Integration Patterns

### Standard Usage
A world generation worker receives a pre-configured EnvironmentProvider instance from its parent orchestrator. For each block position it processes, it creates a `Context` and queries the provider to influence its logic.

```java
// Executed within a world generation worker thread
EnvironmentProvider temperatureProvider = generatorConfig.getTemperatureProvider();
Vector3i currentPosition = new Vector3i(x, y, z);
WorkerIndexer.Id workerId = this.getWorkerId();

// Create a context for the query
EnvironmentProvider.Context queryContext = new EnvironmentProvider.Context(currentPosition, workerId);

// Sample the environment
int temperature = temperatureProvider.getValue(queryContext);

// Use the value in subsequent logic
if (temperature > SNOW_THRESHOLD) {
    // Place grass instead of snow
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Never modify the internal state of an EnvironmentProvider within the `getValue` method. This will break determinism and thread safety. All state should be injected at construction time.
-   **Direct Instantiation in Workers:** Do not use `new SomeNoiseProvider()` inside a worker. This will result in each worker using a different instance, potentially with different seeds, causing visible seams between world chunks. Always use the shared instance provided by the generator's configuration.
-   **Ignoring the Context:** While some simple providers may not use the `workerId`, it exists for more advanced implementations that might leverage thread-local storage or other worker-specific optimizations. Do not assume it can be null.

## Data Pipeline
The EnvironmentProvider acts as a data source early in the block placement pipeline for a given coordinate.

> Flow:
> World Generation Worker -> Creates `Context(position, workerId)` -> **EnvironmentProvider.getValue(Context)** -> Returns `int` value -> Biome/Block Selection Logic -> Final Block Placement


---
description: Architectural reference for BiCarta
---

# BiCarta<R>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Framework Contract

## Definition
```java
// Signature
public abstract class BiCarta<R> {
```

## Architecture & Concepts
BiCarta is an abstract base class that defines a fundamental contract within the Hytale world generation framework. It represents a deterministic, two-dimensional sampling function, primarily used to query data at specific world coordinates. The name implies its function: a "Bi" (two-input) "Carta" (map or chart).

This class acts as a functional interface for any generator stage that needs to produce a value based on an (X, Z) coordinate pair. Implementations of BiCarta are central to procedural generation, powering systems like biome mapping, elevation sampling, and moisture distribution.

The inclusion of a `WorkerIndexer.Id` in the `apply` method signature is a critical architectural feature. It signals that the world generator is a massively parallel system. This parameter allows implementations to be aware of the specific worker thread that is invoking them, enabling thread-local caching, deterministic seeding per-thread, or other advanced concurrency patterns without relying on expensive global locks.

Furthermore, the `allPossibleValues` method mandates that the output domain of any BiCarta function is a finite, enumerable set. This constraint is leveraged by the generator for validation, optimization, and pre-allocation of resources.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of BiCarta are not instantiated ad-hoc. They are constructed and configured by higher-level generator services or factories during the initialization of a world generation pipeline. They are effectively part of a static, pre-built generation plan.
- **Scope:** An instance of a BiCarta implementation is typically long-lived, persisting for the entire duration of a world generation session. They are designed to be stateless and are therefore reusable across the entire generation task.
- **Destruction:** Ownership is managed by the parent generator or service that created the instance. They are subject to standard Java garbage collection once the world generation session is complete and all references are released.

## Internal State & Concurrency
- **State:** The BiCarta contract is stateless. Concrete implementations are **strongly expected to be immutable**. Any configuration, such as seeds or noise parameters, must be provided during construction and must not be modified during the component's lifecycle. Maintaining immutability is essential for ensuring the generator's output is deterministic and repeatable.
- **Thread Safety:** Implementations **must be thread-safe**. The `apply` method is guaranteed to be invoked concurrently from multiple worker threads. The design explicitly avoids the need for locks by providing the `WorkerIndexer.Id`, encouraging thread-local computation. Any deviation from a stateless, immutable design will introduce severe race conditions and non-deterministic behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(int, int, WorkerIndexer.Id) | R | Varies | The core sampling function. Returns a value of type R for a given 2D coordinate and worker context. |
| allPossibleValues() | List<R> | O(1) | Returns a pre-computed list of every possible value that can be returned by `apply`. |

## Integration Patterns

### Standard Usage
BiCarta is not used directly but is implemented by concrete classes. A generator system will hold a reference to an implementation and invoke it repeatedly within its core processing loop.

```java
// A generator system obtains a pre-configured BiCarta instance
// (e.g., a BiomeCarta for biome placement).
BiomeCarta biomeMap = generatorContext.getBiomeCarta();
WorkerIndexer.Id workerId = currentThread.getWorkerId();

// For each column in the assigned chunk...
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        // Sample the biome at the world coordinate.
        Biome biome = biomeMap.apply(worldX + x, worldZ + z, workerId);
        // ... use the biome to generate terrain.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Mutable Implementations:** Do not create implementations that modify their internal state within the `apply` method. This violates the core principle of deterministic generation and will lead to unpredictable results and race conditions.
- **Expensive Computation in `allPossibleValues`:** The `allPossibleValues` method should return a cached, static list. Calculating the list on each call is inefficient and defeats its purpose as a fast lookup mechanism.
- **Ignoring Thread Safety:** Assuming `apply` will be called serially is incorrect. All implementations must be prepared for concurrent invocation from dozens of threads.

## Data Pipeline
BiCarta acts as a data source within the broader generation pipeline. It does not process incoming data; it creates data based on spatial inputs.

> Flow:
> World Generator requests data for coordinate (X, Z) -> **BiCarta.apply(X, Z, workerId)** -> Returns value R -> World Generator uses R to build world data (e.g., set biome, determine block type)


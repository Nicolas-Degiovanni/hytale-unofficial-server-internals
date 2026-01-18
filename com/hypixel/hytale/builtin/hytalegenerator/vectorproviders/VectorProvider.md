---
description: Architectural reference for VectorProvider
---

# VectorProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.vectorproviders
**Type:** Abstract Component

## Definition
```java
// Signature
public abstract class VectorProvider {
```

## Architecture & Concepts

The VectorProvider is an abstract base class that forms a core component of the procedural world generation system. It embodies a Strategy Pattern, defining a contract for any component that needs to compute a Vector3d value based on a specific world-space context.

Its primary role is to act as a modular, pluggable function in the world generation pipeline. Concrete implementations of VectorProvider are used to introduce spatial variance and complex transformations. Common use cases include:

*   Applying 3D noise to offset terrain features.
*   Defining flow fields for biome distribution or river generation.
*   Creating constant vectors for uniform transformations.
*   Composing multiple providers to build sophisticated mathematical effects.

The system's power lies in its composition. Higher-level generator components, such as Density functions, do not implement vector logic themselves. Instead, they delegate this responsibility to a configured VectorProvider instance, promoting reusability and declarative world design.

The provided **Context** object is the key data structure that decouples the provider from the caller. It supplies all necessary environmental information, such as the sample position and worker thread ID, ensuring that provider implementations can be pure, deterministic, and thread-safe.

### Lifecycle & Ownership
-   **Creation:** Concrete VectorProvider implementations are not typically instantiated directly in code. They are deserialized from world generation asset files (e.g., JSON) by the Hytale Generator framework during the loading of a world generation configuration.
-   **Scope:** An instance of a VectorProvider is stateless and immutable after creation. It persists for as long as the world generation configuration is held in memory, effectively acting as a singleton template for a specific vector calculation.
-   **Destruction:** Instances are eligible for garbage collection when the world generator unloads its current configuration, for example, when a server shuts down or a different world is loaded.

## Internal State & Concurrency
-   **State:** The VectorProvider base class is stateless. Concrete implementations are designed to be **immutable**. Their configuration (e.g., noise frequency, seed, constant vector value) is set upon creation and must not change during their lifetime. All per-request data is passed via the Context object.
-   **Thread Safety:** Implementations of this class **must be thread-safe**. The world generation system processes chunks across multiple worker threads concurrently. The same VectorProvider instance will be invoked simultaneously by different threads with different Context objects. The inclusion of WorkerIndexer.Id in the Context facilitates deterministic, thread-aware calculations without requiring locks or mutable shared state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | Vector3d | Varies | Computes and returns a Vector3d based on the provided context. This is the core computational method. Complexity is dependent on the concrete implementation. |

## Integration Patterns

### Standard Usage

A developer does not typically invoke a VectorProvider directly. It is configured as part of a larger system, such as a Density function, which then calls the process method during its own evaluation.

```java
// Hypothetical usage within a higher-level generator component
// Note: 'this.configuredProvider' would be loaded from an asset file.

public class WarpedDensity implements Density {
    private VectorProvider configuredProvider;

    @Override
    public double get(Context densityContext) {
        // 1. Create a context for the provider
        VectorProvider.Context vecContext = new VectorProvider.Context(densityContext);

        // 2. Invoke the provider to get a displacement vector
        Vector3d offset = this.configuredProvider.process(vecContext);

        // 3. Use the result to modify the original position
        Vector3d warpedPosition = densityContext.position.add(offset);

        // 4. Continue calculation with the new position
        return SomeNoiseFunction.get(warpedPosition);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Mutable State:** Implementing a subclass that modifies its own instance fields within the process method is a critical error. This will break thread safety and lead to severe, non-deterministic generation bugs.
-   **Ignoring Context:** Failing to use the position from the input Context will result in a constant output, defeating the purpose of the provider. Ignoring the workerId may prevent access to thread-specific resources where applicable.
-   **Manual Instantiation:** Avoid constructing providers with *new*. They are designed to be managed and configured by the declarative world generation framework.

## Data Pipeline

The VectorProvider acts as a functional step within a larger data processing pipeline, typically initiated by a request to calculate terrain density at a specific point.

> Flow:
> World Generator requests density at (x,y,z) -> Density Function is invoked -> Creates **VectorProvider.Context** -> **VectorProvider.process()** is called -> Returns displacement Vector3d -> Density Function uses the vector to sample noise at a new position -> Final density value is returned.


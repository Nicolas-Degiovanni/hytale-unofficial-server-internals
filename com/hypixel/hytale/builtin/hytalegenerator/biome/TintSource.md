---
description: Architectural reference for TintSource
---

# TintSource

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface TintSource {
```

## Architecture & Concepts
The TintSource interface defines a strict contract for any object that can supply a TintProvider. Its primary architectural role is to decouple systems that *require* color tinting information (such as biome renderers or foliage shaders) from the concrete sources of that information (like Biome definitions or specific block states).

This abstraction is a key component of the world generation and rendering pipeline, allowing for a flexible and extensible system where various game objects can influence the final coloration of the environment without requiring the rendering engine to have explicit knowledge of each object type. It embodies the "program to an interface, not an implementation" design principle.

## Lifecycle & Ownership
As an interface, TintSource itself does not have a lifecycle. The lifecycle, scope, and ownership are dictated entirely by the class that implements this contract.

- **Creation:** An object implementing TintSource is typically created as part of a larger system's initialization. For example, a Biome object, which implements TintSource, is loaded from configuration files by the WorldGenerator service.
- **Scope:** The scope of an implementing object varies. A Biome instance is typically a long-lived, session-scoped singleton. Other implementations might be transient, existing only for a single frame or calculation.
- **Destruction:** Implementing objects are managed by their respective owners. Session-scoped objects are cleaned up during world or client shutdown.

**WARNING:** Systems consuming a TintSource must not make assumptions about its lifecycle. Always retrieve a valid instance from the appropriate registry or context for the current operation.

## Internal State & Concurrency
The TintSource interface is stateless. It defines a single, zero-argument method and holds no internal data.

- **State:** Responsibility for state management (e.g., caching the TintProvider) lies entirely with the implementing class.
- **Thread Safety:** The contract itself is inherently thread-safe. However, the thread safety of the `getTintProvider` method is **not guaranteed** and depends on the specific implementation. Consumers must refer to the documentation of the concrete class to understand its concurrency guarantees.

**WARNING:** Calling `getTintProvider` from multiple threads may be unsafe if the underlying implementation is mutable and not properly synchronized. This is a common source of race conditions during parallelized chunk generation.

## API Surface
The public contract consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTintProvider() | TintProvider | O(1) | Retrieves the TintProvider associated with this source. Implementations should not perform heavy computation here. |

## Integration Patterns

### Standard Usage
The intended pattern is to treat any object that implements TintSource as an opaque provider of tinting information. The consumer should be concerned only with obtaining the TintProvider, not with the underlying type of the TintSource.

```java
// biome is an object that implements TintSource
TintProvider provider = biome.getTintProvider();
int color = provider.getTintColor(world, position);
// ... apply color to a mesh
```

### Anti-Patterns (Do NOT do this)
- **Type Checking:** Do not use `instanceof` to check the concrete type of a TintSource to alter behavior. This violates the principle of the interface and creates brittle, hard-to-maintain code.

    ```java
    // BAD: Violates the abstraction
    if (source instanceof ForestBiome) {
        // ... special logic for forests
    }
    ```

- **Assuming Implementation Details:** Do not assume the returned TintProvider is of a specific subclass. Always program against the base TintProvider interface.

## Data Pipeline
TintSource acts as a factory or accessor within a larger data flow, typically initiated by a rendering or world-generation request.

> Flow:
> World Generator/Renderer -> Requests Biome data -> Obtains object implementing **TintSource** -> Calls `getTintProvider()` -> Uses TintProvider to calculate color -> Applies color to vertex data


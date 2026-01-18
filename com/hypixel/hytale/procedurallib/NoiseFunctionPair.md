---
description: Architectural reference for NoiseFunctionPair
---

# NoiseFunctionPair

**Package:** com.hypixel.hytale.procedurallib
**Type:** Transient

## Definition
```java
// Signature
public class NoiseFunctionPair implements NoiseFunction {
```

## Architecture & Concepts
The NoiseFunctionPair is a composite object that unifies two-dimensional and three-dimensional noise generation into a single, polymorphic interface. It acts as a specialized container and delegate, holding references to a NoiseFunction2d and a NoiseFunction3d implementation.

Its primary architectural role is to simplify the systems that consume noise data, such as world generators or biome placement algorithms. By conforming to the generic NoiseFunction interface, it allows higher-level logic to request noise values without needing to know whether the underlying source is 2D or 3D. The selection of the appropriate delegate is handled implicitly through method overloading of the *get* method.

This class embodies the **Composite Pattern**, allowing clients to treat individual noise objects and compositions of noise objects uniformly. It is a fundamental building block for constructing complex, multi-dimensional noise graphs from configuration files or procedural algorithms.

### Lifecycle & Ownership
- **Creation:** NoiseFunctionPair instances are typically created during the initialization phase of a procedural generation process. They are constructed either directly or by a factory responsible for parsing a world generation configuration. They are not managed by a central service registry.
- **Scope:** The lifetime of a NoiseFunctionPair is bound to the specific generation task it supports. For example, an instance may be created for a single chunk generation and discarded afterward. It is not a long-lived, session-scoped object.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are dropped, typically upon completion of the generation task that owns it. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The state of this class is **highly mutable**. The internal references to the 2D and 3D noise functions can be swapped at any time via public setters. The object itself is a stateful container and does not perform any caching of noise values; all calls are passed directly to the underlying delegates.

- **Thread Safety:** This class is **not thread-safe**. The setter methods are unsynchronized. If one thread calls a setter while another thread is executing a *get* method, a race condition will occur. This can lead to unpredictable behavior, including NullPointerExceptions if a function is set to null during a read.

    **WARNING:** All access and mutation of a NoiseFunctionPair instance must be externally synchronized or confined to a single thread. It is not safe for concurrent use in multi-threaded world generation without explicit locking.

## API Surface
The public API is designed for configuration and delegation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offset, x, y) | double | O(N) | Delegates the call to the internal NoiseFunction2d. Throws NullPointerException if the 2D function is not set. |
| get(seed, offset, x, y, z) | double | O(N) | Delegates the call to the internal NoiseFunction3d. Throws NullPointerException if the 3D function is not set. |
| setNoiseFunction2d(func) | void | O(1) | Sets or replaces the internal 2D noise function delegate. |
| setNoiseFunction3d(func) | void | O(1) | Sets or replaces the internal 3D noise function delegate. |

*Complexity O(N) denotes that performance is dependent on the complexity of the delegated noise function.*

## Integration Patterns

### Standard Usage
The intended use is to configure the pair with concrete noise implementations and then pass the pair object to a system that operates on the generic NoiseFunction interface.

```java
// 1. Obtain concrete 2D and 3D noise implementations
NoiseFunction2d perlin2d = new PerlinNoise2d(...);
NoiseFunction3d simplex3d = new SimplexNoise3d(...);

// 2. Create and configure the pair
NoiseFunctionPair noisePair = new NoiseFunctionPair();
noisePair.setNoiseFunction2d(perlin2d);
noisePair.setNoiseFunction3d(simplex3d);

// 3. Pass the unified interface to a consumer
WorldGenerator generator = new WorldGenerator(noisePair);
generator.generateTerrain(); // Internally calls noisePair.get(...)
```

### Anti-Patterns (Do NOT do this)
- **Partial Initialization:** Instantiating the object and calling a *get* method before the corresponding delegate has been set is a critical error. This will always result in a NullPointerException.

    ```java
    // BAD: The 2D function is not set
    NoiseFunctionPair incompletePair = new NoiseFunctionPair();
    incompletePair.setNoiseFunction3d(new SimplexNoise3d());
    double value = incompletePair.get(seed, 0, 10.0, 20.0); // Throws NullPointerException
    ```

- **Concurrent Modification:** Modifying the internal delegates from one thread while another thread is actively using the pair for noise generation is unsafe and will lead to race conditions or exceptions.

    ```java
    // BAD: Unsafe concurrent modification
    NoiseFunctionPair sharedPair = ...;
    
    // Thread 1
    new Thread(() -> {
        while(true) { sharedPair.get(seed, 0, x, y, z); }
    }).start();

    // Thread 2
    new Thread(() -> {
        // This can cause a crash or inconsistent state in Thread 1
        sharedPair.setNoiseFunction3d(newDifferentNoise());
    }).start();
    ```

## Data Pipeline
NoiseFunctionPair acts as a routing node within a larger data flow. It does not originate or terminate data but directs requests based on method signature.

> Flow:
> World Generator -> NoiseFunction.get(x, y) -> **NoiseFunctionPair** -> NoiseFunction2d.get(x, y) -> Return Value


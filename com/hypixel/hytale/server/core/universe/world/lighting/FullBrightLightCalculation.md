---
description: Architectural reference for FullBrightLightCalculation
---

# FullBrightLightCalculation

**Package:** com.hypixel.hytale.server.core.universe.world.lighting
**Type:** Decorator

## Definition
```java
// Signature
public class FullBrightLightCalculation implements LightCalculation {
```

## Architecture & Concepts
The FullBrightLightCalculation class is an implementation of the **Decorator** design pattern. It wraps another, more complex LightCalculation implementation (the *delegate*) to enforce a simplistic, high-performance lighting model where every block in a chunk section is fully illuminated by sky light.

Its primary role is to act as a post-processing step in the lighting pipeline. It first allows its delegate to perform the standard lighting computation. Immediately after the delegate completes, this class discards the result and overwrites the entire chunk section's light map with a maximum sky light value of 15.

This component is not a general-purpose lighting engine. It is a strategic override, likely enabled by server configuration for specific world types where complex, dynamic lighting is undesirable. Common use cases include:
*   Creative mode worlds where visibility is paramount.
*   Server lobbies or minigames where performance is prioritized over visual fidelity.
*   Worlds with a disabled day-night cycle.

By delegating first, it ensures that the underlying world state (e.g., heightmaps) managed by the delegate is correctly updated, even though the final light values are ultimately ignored and replaced.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level factory or manager, such as the ChunkLightingManager, during the world's initialization phase. It is constructed with a reference to a concrete LightCalculation delegate, which it then wraps.
- **Scope:** The object's lifetime is bound to the active lighting strategy of the world. It persists as long as the server configuration dictates a "full bright" lighting model. If the lighting strategy is changed at runtime, this object and its delegate would be discarded.
- **Destruction:** Marked for garbage collection when the ChunkLightingManager is destroyed or reconfigured with a different LightCalculation implementation. It performs no explicit resource cleanup.

## Internal State & Concurrency
- **State:** This class is stateful, as it holds immutable references to the ChunkLightingManager and its delegate LightCalculation. It does not maintain any other mutable internal state. All modifications it performs are on the WorldChunk and BlockSection objects passed to it as method arguments.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. Concurrency control is the strict responsibility of the caller, which is expected to be a single-threaded lighting calculation queue or a system that guarantees exclusive write access to a WorldChunk during processing.

    **Warning:** Unsynchronized, concurrent calls to its methods will lead to severe race conditions, data corruption in chunk light maps, and client-side visual artifacts. All interactions must be serialized by the owning ChunkLightingManager.

## API Surface
The public API conforms to the LightCalculation interface, with the addition of the public `setFullBright` utility method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateLight(chunkPosition) | CalculationResult | O(N) | Delegates calculation, then overwrites the entire section's light map. N is the number of blocks in a section (32768). |
| invalidateLightAtBlock(...) | boolean | O(N) | Delegates invalidation, then overwrites the entire section's light map if the delegate handled the event. |
| invalidateLightInChunkSections(...) | boolean | O(M * N) | Delegates invalidation, then overwrites light maps for a range of M sections. |
| setFullBright(worldChunk, chunkY) | void | O(N) | Core implementation. Directly sets the sky light of every block in a section to 15 and flags the section for a network update. |

## Integration Patterns

### Standard Usage
This class should never be used directly by game logic. It is configured within the server's world management systems to wrap a standard lighting engine.

```java
// Example of how ChunkLightingManager might configure the lighting pipeline
LightCalculation standardEngine = new StandardLightCalculation(world);
LightCalculation fullBrightOverride = new FullBrightLightCalculation(this, standardEngine);

// The manager now uses the decorator for all operations
this.activeLightCalculation = fullBrightOverride;

// Later, a chunk needs lighting...
activeLightCalculation.calculateLight(chunkPos);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually construct this class in game systems. The lighting pipeline is determined by server or world configuration and managed centrally. Manually creating an instance can lead to a desynchronized lighting state.

- **Incorrect Chaining:** This decorator should be the final, outermost layer in the lighting chain. Wrapping it with another decorator is almost always a mistake, as FullBrightLightCalculation erases any prior lighting work.
    ```java
    // BAD: The work of SomeOtherDecorator is erased
    LightCalculation badChain = new SomeOtherDecorator(new FullBrightLightCalculation(...));
    ```

- **Ignoring Delegate Result:** While this class overwrites the delegate's light values, it relies on the delegate to correctly update other world state, like heightmaps. Bypassing the delegate call and calling `setFullBright` directly can corrupt world data.

## Data Pipeline
The primary function of this class is to intercept the data flow from a standard lighting engine and replace the calculated light map with a static, fully-lit version before it is committed to the chunk and sent to clients.

> Flow:
> World Change (e.g., Block Placement) -> ChunkLightingManager schedules update -> `delegate.calculateLight()` computes realistic lighting -> **FullBrightLightCalculation** discards result -> `setFullBright()` generates a new, max-value light map -> `BlockSection.setGlobalLight()` commits the map -> `BlockChunk.invalidateChunkSection()` flags for network sync -> Server sends updated chunk section to clients.


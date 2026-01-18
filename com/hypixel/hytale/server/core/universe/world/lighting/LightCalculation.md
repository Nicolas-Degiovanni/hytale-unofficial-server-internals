---
description: Architectural reference for LightCalculation
---

# LightCalculation

**Package:** com.hypixel.hytale.server.core.universe.world.lighting
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface LightCalculation {
   void init(@Nonnull WorldChunk var1);

   @Nonnull
   CalculationResult calculateLight(@Nonnull Vector3i var1);

   boolean invalidateLightAtBlock(@Nonnull WorldChunk var1, int var2, int var3, int var4, @Nonnull BlockType var5, int var6, int var7);

   boolean invalidateLightInChunkSections(@Nonnull WorldChunk var1, int var2, int var3);
}
```

## Architecture & Concepts
The LightCalculation interface defines the abstract contract for all server-side light propagation algorithms. It serves as a critical component of the world simulation engine, responsible for modeling how light (both from the sky and from luminous blocks) spreads, attenuates, and is occluded by terrain.

This interface employs a **Strategy Pattern**, decoupling the WorldChunk entity from the concrete implementation of any specific lighting model. A WorldChunk object will hold and delegate to one or more implementations of LightCalculation (e.g., one for skylight, one for block light) to manage its lighting state. This allows the engine to potentially swap or modify lighting behaviors without altering the core chunk management logic.

Its primary responsibility is to manage a queue of "dirty" light nodes and process them, propagating light changes outward from a source until a steady state is reached.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of this interface are instantiated by the world generation system when a new WorldChunk is created or loaded from storage. Each chunk typically owns its own dedicated instances of light calculators.
- **Scope:** The lifecycle of a LightCalculation implementation is strictly bound to its owning WorldChunk. It persists in memory for as long as the chunk is active on the server.
- **Destruction:** The object is marked for garbage collection when its parent WorldChunk is unloaded from the server's active memory. There are no external persistent references.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, any concrete implementation is expected to be highly stateful and mutable. Implementations will maintain internal data structures, such as priority queues for light propagation and caches for previously computed light values, to optimize performance. This state is a direct representation of the lighting data for the associated chunk.
- **Thread Safety:** Implementations of LightCalculation are **not thread-safe**. All method calls on a given instance must be synchronized externally. Typically, all world modifications, including lighting updates for a specific region, are processed sequentially on a single, dedicated world-update thread to prevent race conditions and ensure data consistency within a chunk.

**WARNING:** Accessing a LightCalculation instance from multiple threads without explicit locking will lead to catastrophic world corruption, including inconsistent lighting, deadlocks, and server crashes.

## API Surface
The public contract is designed for coarse-grained operations orchestrated by a WorldChunk.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(WorldChunk) | void | O(N³) | Initializes the calculator for a given chunk, typically performing a full, top-down scan for initial skylight. This is a blocking, high-cost operation. |
| calculateLight(Vector3i) | CalculationResult | Varies | Returns the calculated light values at a specific block coordinate. May be an O(1) cache lookup or trigger a wider recalculation. |
| invalidateLightAtBlock(...) | boolean | O(k) | Marks a single block's light as invalid, adding it to an internal queue for recalculation. The complexity depends on the number of neighbors (k) affected. |
| invalidateLightInChunkSections(...) | boolean | O(N²) | Marks entire vertical sections of a chunk as invalid. Used for large-scale changes, such as loading adjacent chunks. |

## Integration Patterns

### Standard Usage
The interface is intended to be used by the WorldChunk class. When a block is placed or removed, the chunk delegates to its LightCalculation instance to invalidate the affected area. The actual recalculation is deferred and processed in a batch during the server's world-tick cycle.

```java
// Within a WorldChunk method after a block change
// this.skylightCalculator is a concrete implementation of LightCalculation
boolean needsUpdate = this.skylightCalculator.invalidateLightAtBlock(this, x, y, z, newBlock, ...);

if (needsUpdate) {
    // The world's scheduler will later process the update queue
    // for this chunk's lighting system.
    world.getLightingManager().scheduleUpdate(this);
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instances:** Never share a single LightCalculation instance between multiple WorldChunk objects. Each calculator's internal state is intrinsically tied to one and only one chunk.
- **External Instantiation:** Do not create instances of LightCalculation implementations directly. The world engine is responsible for creating and binding them to chunks to ensure correct initialization.
- **Synchronous Recalculation:** Avoid designs that call invalidateLightAtBlock and then immediately attempt to read the new value with calculateLight. The system is designed to be asynchronous, with invalidations being queued and processed in batches for performance.

## Data Pipeline
The flow of data for a lighting update is managed by the world tick and chunk systems.

> Flow:
> Player Action (Block Placement) -> Server Tick -> WorldChunk.setBlock() -> **LightCalculation.invalidateLightAtBlock()** -> World Lighting Manager Queue -> **(Deferred) Recalculation Task** -> WorldChunk Data Array Update -> Network Serialization -> Client Render


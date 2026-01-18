---
description: Architectural reference for DirectionalGrowthBehaviour
---

# DirectionalGrowthBehaviour

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages.spread
**Type:** Configuration Object

## Definition
```java
// Signature
public class DirectionalGrowthBehaviour extends SpreadGrowthBehaviour {
```

## Architecture & Concepts
The DirectionalGrowthBehaviour is a data-driven component that defines a specific algorithm for block spreading within the Hytale farming system. It represents a concrete implementation of a SpreadGrowthBehaviour, focusing on placing new blocks at a random offset from a source block, constrained by configured horizontal and vertical ranges.

Architecturally, this class serves as a bridge between the high-level farming configuration (defined in asset files) and the low-level world manipulation APIs. It encapsulates the complex logic of selecting a target location, validating the placement against world geometry and physics rules, and safely modifying the world state.

Its primary role is to enable content designers to create diverse and organic growth patterns (e.g., vines, moss, crystal clusters) without writing new code. This is achieved through its static CODEC, which deserializes its behavior from a structured data format, effectively treating the logic as a configurable asset. The class heavily interacts with core server systems, including the ChunkStore for world data access and the BlockPhysics system for placement validation.

## Lifecycle & Ownership
- **Creation:** Instances of DirectionalGrowthBehaviour are not created directly using the constructor. They are instantiated by the server's asset loading pipeline via the static BuilderCodec named CODEC. This process occurs when the server loads game assets, such as farming definitions, from JSON files.
- **Scope:** An instance persists for the lifetime of the server session, as it is part of a larger, immutable asset configuration. It is effectively a read-only data object after its initial creation.
- **Destruction:** The object is eligible for garbage collection when its parent asset configuration is unloaded, which typically only happens during a server shutdown or a full asset reload.

## Internal State & Concurrency
- **State:** The internal state, including blockTypes, horizontalRange, and verticalRange, is populated once during deserialization. After this point, the state is treated as **immutable**. The execute method reads this configuration but does not modify it.
- **Thread Safety:** The class is inherently thread-safe for read operations due to its immutable state. The primary method, execute, is designed to be called from systems that may operate on different threads. It performs its initial, potentially expensive, calculations (position selection, validation) synchronously. However, the final, world-modifying operation is asynchronously scheduled on the main world thread via `world.execute`. This is a critical safety mechanism to prevent race conditions and ensure all world modifications are serialized.

**WARNING:** While the initial calculations are safe to run off the main thread, the actual block placement is deferred. Callers should not assume the block exists immediately after the execute method returns.

## API Surface
The public API is minimal, with the execute method serving as the sole entry point for triggering the behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(N) | Attempts to spread a new block. N is the constant PLACE_BLOCK_TRIES (100). It calculates a target position, validates it, and schedules a block placement on the main world thread. This operation is not guaranteed to succeed. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay programmers. It is configured within an asset file and invoked by the underlying farming or block-tick systems. The primary interaction is through data, not code.

A hypothetical invocation by the engine might look like this:
```java
// Engine-level code (e.g., in a farming system)
// This behaviour instance would be retrieved from a loaded asset configuration.
DirectionalGrowthBehaviour behaviour = farmingStage.getGrowthBehaviour();

// The engine calls execute during a block update tick.
behaviour.execute(commandBuffer, sectionRef, blockRef, worldX, worldY, worldZ, newSpreadRate);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DirectionalGrowthBehaviour()`. This will result in a non-functional object with null fields, as it bypasses the CODEC-based deserialization and configuration process.
- **State Mutation:** Do not attempt to modify the internal state of a configured instance after it has been loaded. This violates its design as an immutable configuration object and can lead to unpredictable behavior across the server.
- **Assuming Synchronous Placement:** Do not write code that assumes the target block is placed immediately after `execute` returns. The placement is scheduled asynchronously and will occur at a later point in the world's tick cycle.

## Data Pipeline
The data flow for a single execution of this behavior involves multiple stages, transitioning from a system-level trigger to a direct world modification.

> Flow:
> Farming System Tick -> **DirectionalGrowthBehaviour.execute()** -> Random Position Calculation -> World Chunk Access -> BlockPhysics Validation -> `world.execute(lambda)` -> Main Thread Execution -> `WorldChunk.placeBlock()` -> `FarmingBlock` Component Update


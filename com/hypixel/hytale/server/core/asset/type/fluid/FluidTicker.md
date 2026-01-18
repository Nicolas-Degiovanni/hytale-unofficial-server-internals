---
description: Architectural reference for FluidTicker
---

# FluidTicker

**Package:** com.hypixel.hytale.server.core.asset.type.fluid
**Type:** Data-Driven Strategy Component

## Definition
```java
// Signature
public abstract class FluidTicker {
```

## Architecture & Concepts

The FluidTicker is an abstract base class that forms the core of the server-side fluid simulation engine. It embodies a **Strategy Pattern**, where each concrete implementation (e.g., WaterTicker, LavaTicker) defines the unique physical behavior of a specific fluid type. This architecture decouples the general simulation loop from the specific rules governing how a fluid flows, spreads, or dissipates.

This system is fundamentally **data-driven**. FluidTicker instances are not instantiated directly in code via constructors. Instead, they are deserialized from game asset files using the Hytale **Codec** system. This allows designers to define new fluids and modify properties like flow rate or source block requirements purely through configuration, without requiring changes to the core engine code.

A FluidTicker operates within the server's block update mechanism, commonly known as the **Block Tick System**. A fluid block is not processed on every single game tick. Instead, the `tick` method is invoked only when the block is scheduled for an update. The method contains further throttling logic based on the configured `flowRate` and the world's current tick, distributing the computational load of fluid simulation across many game ticks to maintain server performance.

The primary responsibility of a FluidTicker is to execute the state transition logic for a single fluid block. It reads the state of the block and its neighbors, determines the next state, and applies the change to the world.

### Lifecycle & Ownership
- **Creation:** Instances are created by the asset loading system at server startup. The `CodecMapCodec` deserializes fluid definitions from asset files and constructs the corresponding FluidTicker object for each defined fluid type.
- **Scope:** Application-scoped. Once loaded, a single instance of a specific FluidTicker (e.g., the one for water) is shared and reused for all fluid blocks of that type across the entire world. They persist for the lifetime of the server.
- **Destruction:** Instances are garbage collected when the server shuts down and the central asset registry is cleared.

## Internal State & Concurrency
- **State:** A FluidTicker instance is effectively immutable after its initial creation from asset data. Its configuration, such as `flowRate` and `canDemote`, is fixed. The class is designed to be stateless regarding the world; all necessary world state (fluid levels, block types) is passed as arguments to its methods. The `supportedById` field is a lazily-initialized cache of an asset ID, a thread-safe optimization.

- **Thread Safety:** This class is designed to be thread-safe and is frequently called from multiple worker threads processing different parts of the world simultaneously.
    - Its methods operate as pure functions on the provided world data.
    - It does not modify its own internal fields during a tick operation.
    - The provided `CachedAccessor` utilizes a `ThreadLocal` instance, ensuring that chunk data caches are isolated to the currently executing thread, preventing data corruption and race conditions.

    **WARNING:** Concrete implementations must be stateless. Storing per-block or per-tick information in instance fields will lead to severe concurrency bugs and unpredictable simulation behavior.

## API Surface

The public API provides the entry points for the server's block tick scheduler to drive the fluid simulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | BlockTickStrategy | O(1) | Primary entry point. Throttles execution based on `flowRate` and dispatches to `process`. |
| process(...) | BlockTickStrategy | O(1) | Executes the core state transition logic by checking `isAlive` and then calling `spread` or demoting the fluid. |
| isAlive(...) | FluidTicker.AliveStatus | O(1) | Determines if a fluid block should persist based on its neighbors. Checks for sources above or adjacent. |
| spread(...) | abstract | Varies | Abstract method for implementing fluid-specific flow and spread behavior. |
| setTickingSurrounding(...) | static void | O(1) | Wakes up all 26 neighbors of a block, scheduling them for a future tick. Critical for propagating fluid changes. |

## Integration Patterns

### Standard Usage

A FluidTicker is never invoked directly. It is driven by the server's block update system. The system retrieves the correct ticker for a given fluid ID and passes the required world context.

```java
// Simplified example of how the BlockTickManager would use a FluidTicker

// 1. Get the fluid asset for the block being ticked
Fluid fluid = Fluid.getAssetMap().getAsset(fluidId);
FluidTicker ticker = fluid.getTicker();

// 2. Create a thread-local cached accessor for efficient world access
FluidTicker.CachedAccessor accessor = FluidTicker.CachedAccessor.of(commandBuffer, fluidSection, blockSection, 5);

// 3. Invoke the tick method
BlockTickStrategy result = ticker.tick(
    commandBuffer,
    accessor,
    fluidSection,
    blockSection,
    fluid,
    fluidId,
    worldX,
    worldY,
    worldZ
);

// The system then acts on the result (e.g., puts the block to sleep)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WaterTicker()`. The simulation relies on instances configured and loaded by the asset system. Direct instantiation bypasses critical configuration like `flowRate`.
- **Stateful Implementations:** Subclasses must not contain fields that store state related to a specific block or tick. A single ticker instance is shared for thousands of blocks. Storing state on the instance will cause catastrophic simulation errors.
- **Bypassing the Accessor:** Do not read world data from any source other than the provided `Accessor` argument. The `CachedAccessor` is essential for performance and ensures that all data access correctly handles chunk boundaries and loading/unloading.

## Data Pipeline

The flow of data during a single fluid block update is a read-process-write cycle that propagates changes to neighbors.

> Flow:
> Block Tick Scheduler selects a fluid block at (X, Y, Z) -> Retrieve `Fluid` asset and its associated **FluidTicker** -> `tick` method is called -> **FluidTicker** uses `Accessor` to read fluid levels and block types in a 3x3 area -> `isAlive` and `spread` logic calculates the new state for (X, Y, Z) -> **FluidTicker** writes new fluid level/ID to the `FluidSection` -> `setTickingSurrounding` is called, scheduling the 26 neighbors for a future tick.


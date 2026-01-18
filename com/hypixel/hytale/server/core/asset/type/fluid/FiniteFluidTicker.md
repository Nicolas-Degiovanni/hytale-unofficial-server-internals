---
description: Architectural reference for FiniteFluidTicker
---

# FiniteFluidTicker

**Package:** com.hypixel.hytale.server.core.asset.type.fluid
**Type:** Stateless Strategy

## Definition
```java
// Signature
public class FiniteFluidTicker extends FluidTicker {
```

## Architecture & Concepts
The FiniteFluidTicker is a concrete implementation of the `FluidTicker` strategy pattern. It encapsulates the physical simulation logic for finite fluids, such as water or lava, where volume conservation is a primary concern. Unlike an infinite fluid source, a block of finite fluid that spreads will deplete its source.

This class is a core component of the server's block update system. When the world's ticking scheduler determines a fluid block needs to be processed, it delegates the simulation logic to the specific FluidTicker instance associated with that fluid type. The FiniteFluidTicker's responsibility is to calculate how the fluid should flow into adjacent blocks for a single game tick, based on a clear set of physical rules.

The core simulation logic prioritizes downward flow due to gravity. If downward flow is not possible, it attempts to spread sideways. A key architectural feature is its handling of chunk boundaries; the simulation will pause and reschedule itself if it needs to interact with an adjacent chunk that is not yet loaded, preventing server stalls and ensuring data consistency. The return value, a BlockTickStrategy, is the mechanism by which this class communicates the outcome of the tick back to the world scheduler, influencing whether the block should be ticked again immediately, put to sleep, or wait for world state changes.

## Lifecycle & Ownership
- **Creation:** An instance of FiniteFluidTicker is not created manually. It is deserialized and instantiated by the server's asset loading system via its static CODEC field when fluid assets are loaded at server startup. A single instance is created for each type of finite fluid defined in the game's assets.
- **Scope:** Application-scoped. The singleton instance for a given fluid type persists for the entire server session.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and the central asset registries are cleared.

## Internal State & Concurrency
- **State:** **Stateless**. The FiniteFluidTicker class contains no instance-level state. All calculations are performed on data passed in as method arguments, primarily through the `Accessor` interface which provides a view into the current world state for the tick. Its behavior is deterministic based on its inputs.
- **Thread Safety:** **Conditionally Safe**. As a stateless object, the class is inherently thread-safe. However, it is designed to be executed exclusively by the world's single-threaded block ticking system for a given world region. The methods perform a series of non-atomic reads and writes to chunk data. **WARNING:** Invoking its methods concurrently on the same or adjacent chunk sections from multiple threads will lead to severe data corruption, race conditions, and unpredictable fluid behavior. The parent ticking system is responsible for ensuring serialized access.

## API Surface
The public contract is inherited from the parent FluidTicker. The `spread` method is the primary entry point for the simulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isAlive(...) | FluidTicker.AliveStatus | O(1) | Determines if the fluid block should be considered for ticking. Always returns ALIVE for this implementation. |
| spread(...) | BlockTickStrategy | O(N) | Executes a single simulation step for a fluid block. Orchestrates downward and sideways flow. Returns a strategy for the next tick. |

## Integration Patterns

### Standard Usage
The FiniteFluidTicker is not intended to be invoked directly by game logic developers. It is used internally by the world ticking system. The system retrieves the appropriate ticker from a fluid's asset definition and calls `spread`.

```java
// Conceptual example from a hypothetical WorldTickManager
Fluid fluid = getFluidAssetForBlock(x, y, z);
FluidTicker ticker = fluid.getTicker(); // Returns a FiniteFluidTicker instance

// The accessor provides a safe, transactional view of the world for this tick
FluidTicker.Accessor accessor = createAccessorForTick();
BlockTickStrategy nextAction = ticker.spread(world, tick, accessor, ...);

// The scheduler acts on the result
world.getScheduler().handleStrategy(blockPosition, nextAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new FiniteFluidTicker()`. The instance must be managed by the asset system to ensure the correct simulation logic is tied to the correct fluid type.
- **External Invocation:** Do not call the `spread` method from other game systems. It is tightly coupled to the lifecycle and state guarantees of the block ticking scheduler. Bypassing the scheduler can break fluid conservation and cause cascading world corruption.
- **Stateful Extensions:** Do not extend this class to add state. The stateless design is critical for its reuse across the entire world for a given fluid type.

## Data Pipeline
The flow of data and control for a single fluid block update is as follows. The FiniteFluidTicker is the central processing unit in this flow.

> Flow:
> Block Tick Scheduler identifies active fluid block -> Retrieves `Fluid` asset -> Obtains **FiniteFluidTicker** instance -> Invokes `spread` with world `Accessor` -> **FiniteFluidTicker** reads neighbor block and fluid data -> Calculates new fluid levels -> Writes updated levels to `FluidSection` via `Accessor` -> Returns `BlockTickStrategy` to Scheduler


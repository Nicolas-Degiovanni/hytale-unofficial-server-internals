---
description: Architectural reference for TickProcedure
---

# TickProcedure

**Package:** com.hypixel.hytale.server.core.asset.type.blocktick.config
**Type:** Abstract Strategy

## Definition
```java
// Signature
public abstract class TickProcedure {
```

## Architecture & Concepts
The TickProcedure class is an abstract base for defining the server-side behavior of a block when it receives a simulation tick. It embodies a Strategy Pattern, where concrete implementations define specific actions, such as plant growth, fluid flow, or block decay.

This component is a cornerstone of Hytale's data-driven block system. Rather than hard-coding block behaviors, the engine deserializes TickProcedure configurations from game assets at runtime. The static `CODEC` field is the primary mechanism for this, using a "Type" discriminator to map asset definitions to their corresponding Java implementations.

A TickProcedure's sole responsibility is to execute a unit of logic for a single block at a specific moment. It receives world state as arguments to its `onTick` method and returns a `BlockTickStrategy`. This return value instructs the world's ticking scheduler on how to handle subsequent ticks for that block (e.g., reschedule, stop, change frequency), effectively decoupling the *execution* of a tick from its *scheduling*.

### Lifecycle & Ownership
- **Creation:** TickProcedure instances are not created directly via a constructor. They are instantiated by the server's asset loading pipeline, which uses the static `CODEC` to deserialize them from block configuration files.
- **Scope:** Instances are treated as stateless, immutable singletons for the duration of the server's runtime. A single instance of a specific procedure (e.g., `WheatGrowthProcedure`) is loaded once and shared across all worlds and all instances of the corresponding block type.
- **Destruction:** Instances are garbage collected during server shutdown when the central asset registries are cleared.

## Internal State & Concurrency
- **State:** This class is designed to be completely stateless. All necessary context (World, Chunk, coordinates) is passed as parameters to the `onTick` method. Subclasses are strictly expected to adhere to this stateless contract. Storing per-block or per-world state within a TickProcedure instance is a critical anti-pattern that will lead to severe concurrency bugs.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. The most critical concurrency feature is the `ThreadLocal<SplittableRandom>`. This provides each worker thread in the world simulation engine with its own unique random number generator. This design elegantly avoids the massive performance bottleneck that would arise from multiple threads contending for a lock on a single shared `Random` instance.

## API Surface
The public contract is defined entirely by the abstract `onTick` method, which must be implemented by all concrete subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onTick(World, WorldChunk, int, int, int, int) | BlockTickStrategy | Varies | Executes the defined block behavior. Returns a strategy for the next tick. |

## Integration Patterns

### Standard Usage
Developers do not typically invoke `onTick` directly. Instead, they create a concrete implementation of TickProcedure and register it with the codec system. The world simulation engine then invokes it automatically.

A concrete implementation might look like this:
```java
// A conceptual example of a concrete procedure
public class GrowProcedure extends TickProcedure {
    // Deserialized from asset config
    private final BlockState targetState;
    private final float growthChance;

    // Constructor for deserialization
    public GrowProcedure(BlockState targetState, float growthChance) {
        this.targetState = targetState;
        this.growthChance = growthChance;
    }

    @Override
    public BlockTickStrategy onTick(World world, WorldChunk chunk, int x, int y, int z, int tickCount) {
        if (getRandom().nextFloat() < this.growthChance) {
            world.setBlock(x, y, z, this.targetState);
            return BlockTickStrategy.STOP; // Stop ticking after growth
        }
        return BlockTickStrategy.RESCHEDULE; // Try again later
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not add mutable fields to a TickProcedure subclass that track the state of a specific block in the world. This will break thread-safety and cause unpredictable behavior across the server.
- **Direct Instantiation:** Never use `new MyProcedure()`. The system relies on the asset pipeline and the static `CODEC` for correct initialization and registration.
- **Using Shared Random:** Do not create your own `java.util.Random` instance. Always use the provided `getRandom()` method to acquire the correct thread-local instance, preventing lock contention.

## Data Pipeline
The TickProcedure is a processing stage within the server's world simulation loop. Data flows from the scheduler, through the procedure, and back to the scheduler.

> Flow:
> World Tick Scheduler dequeues a pending tick -> Engine retrieves `World`, `WorldChunk`, and block coordinates -> Engine looks up the block's asset definition -> The associated **TickProcedure** is invoked -> `onTick` executes -> Returns `BlockTickStrategy` -> World Tick Scheduler processes the strategy (reschedules or removes the tick)


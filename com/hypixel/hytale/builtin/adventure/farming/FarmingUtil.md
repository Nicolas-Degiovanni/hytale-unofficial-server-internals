---
description: Architectural reference for FarmingUtil
---

# FarmingUtil

**Package:** com.hypixel.hytale.builtin.adventure.farming
**Type:** Utility

## Definition
```java
// Signature
public class FarmingUtil {
```

## Architecture & Concepts
The FarmingUtil class is a stateless collection of static methods that encapsulates the core logic for the server-side crop and plant growth simulation. It does not represent a persistent system or service; rather, it acts as a pure function library invoked by other engine systems to process farming-related state changes.

Its primary responsibilities are:
1.  **Growth Calculation:** Processing growth ticks for blocks that have a FarmingBlock component. This involves calculating progress based on game time, asset-defined growth durations, and environmental modifiers.
2.  **State Transition:** Applying the results of growth calculations, which may involve changing a block's state, replacing it with a new block, or advancing its growth stage.
3.  **Harvesting Logic:** Handling the consequences of a player or entity harvesting a farmable block, including spawning item drops and potentially resetting the block to a seed stage.

FarmingUtil is a critical link between the world's block ticking mechanism and the data-driven farming configuration defined in game assets. It reads `FarmingData` and `FarmingStageData` assets to determine the rules for growth and harvesting, and writes its results back to the world state via a `CommandBuffer`.

## Lifecycle & Ownership
As a static utility class, FarmingUtil has no instance lifecycle. It is never instantiated, and its methods are globally accessible throughout the server application's lifetime.

-   **Creation:** Not applicable. The class is loaded by the JVM at startup.
-   **Scope:** Application-wide. Its static methods can be called from any context that has access to the class.
-   **Destruction:** Not applicable. The class is unloaded when the JVM shuts down.

Ownership and lifecycle concerns apply to the *data* that FarmingUtil operates on, such as the FarmingBlock component, not the utility class itself.

## Internal State & Concurrency
-   **State:** FarmingUtil is entirely stateless. It contains no mutable fields and all calculations are based exclusively on the arguments passed into its static methods.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the methods operate on mutable, non-thread-safe world state objects like CommandBuffer and BlockSection.

    **WARNING:** The caller is solely responsible for ensuring thread safety. All invocations of FarmingUtil methods must be performed from a context that holds the appropriate locks or guarantees serialized access to the world state, typically the main server tick thread for a given world. Calling these methods from an asynchronous task or a different thread without explicit synchronization will lead to world corruption, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tickFarming(...) | void | O(N) | The primary growth simulation method. Calculates and applies growth progress for a single block over a given time delta. Complexity is O(N) where N is the number of growth stages advanced in a single tick. |
| harvest(...) | void | O(M) | Public entry point for harvesting a farm block. Checks gameplay configuration before delegating to the internal implementation. Complexity is O(M) where M is the number of items in the drop list. |
| generateCapturedNPCMetadata(...) | CapturedNPCMetadata | O(1) | A helper utility to generate NPC metadata from an entity's model component. Its inclusion in this class is incidental. |

## Integration Patterns

### Standard Usage
FarmingUtil is not intended for direct use in typical gameplay scripting. Its methods are invoked by low-level engine systems.

The `tickFarming` method is designed to be called by the server's `BlockSection` ticking scheduler. When a scheduled tick for a farming block is due, the scheduler invokes this method to process its growth.

```java
// Simplified conceptual example of how the engine's block ticker
// would invoke tickFarming.
// This code does not appear in gameplay scripts.

// Inside a world ticking system...
if (blockNeedsFarmingTick(blockRef)) {
    FarmingBlock farmingComponent = getFarmingComponent(blockRef);
    FarmingUtil.tickFarming(
        commandBuffer,
        blockSection,
        sectionRef,
        blockRef,
        farmingComponent,
        x, y, z,
        false
    );
}
```

The `harvest` method is called by the block interaction or destruction systems when a player successfully harvests a crop.

```java
// Simplified conceptual example of a block interaction handler
// calling the harvest logic.

// Inside a block interaction system...
if (interaction.isHarvest() && blockType.hasFarmingData()) {
    FarmingUtil.harvest(
        world,
        entityStoreAccessor,
        playerRef,
        blockType,
        rotation,
        blockPosition
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call `tickFarming` manually in gameplay code. The growth simulation is managed entirely by the engine's scheduled ticking system. Manually calling it will result in incorrect growth rates and desynchronization with game time.
-   **Incorrect Context:** Do not call any method from an asynchronous thread or outside the main world update loop. This will bypass critical state management and synchronization, leading to data corruption.
-   **State Manipulation:** Do not attempt to modify the FarmingBlock component directly to alter growth. All state changes should flow through the `tickFarming` method to ensure game rules, modifiers, and state transitions are correctly applied.

## Data Pipeline
The flow of data through the `tickFarming` method is representative of the class's role as a state processor.

> Flow:
> Scheduled Block Tick Event -> **FarmingUtil.tickFarming** (Reads WorldTime, FarmingData assets, GrowthModifier assets) -> CommandBuffer (Writes component updates, schedules next tick, or removes component) -> World State Update


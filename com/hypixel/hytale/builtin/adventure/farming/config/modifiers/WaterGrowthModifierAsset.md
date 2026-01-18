---
description: Architectural reference for WaterGrowthModifierAsset
---

# WaterGrowthModifierAsset

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.modifiers
**Type:** Configuration Asset

## Definition
```java
// Signature
public class WaterGrowthModifierAsset extends GrowthModifierAsset {
```

## Architecture & Concepts
The WaterGrowthModifierAsset is a data-driven configuration object that implements a specific rule within the server-side farming system. It is not an active, running system but rather a passive data structure with attached logic. Its primary function is to determine if a crop has access to a valid water source and to calculate the corresponding growth speed multiplier.

This asset acts as a bridge between the abstract concept of "watering" and the concrete state of the game world. It defines which specific fluids (e.g., water) and weather conditions (e.g., rain) should be considered valid water sources.

During a block update tick for a plant that uses this modifier, the game engine invokes the `getCurrentGrowthMultiplier` method. This method then queries the surrounding world state—checking for adjacent fluids and current weather—to make its determination. A critical design pattern here is the separation of concerns: this asset modifies the growth of a plant, but it reads and writes state to the `TilledSoilBlock` component on the block *below* the plant. This allows the soil itself to retain state about being watered, which can be used by other systems or for visual effects.

All world state modifications, such as updating the `TilledSoilBlock` component, are funneled through a `CommandBuffer`. This ensures that changes are deferred and executed transactionally at the end of the tick, maintaining world state integrity and determinism.

### Lifecycle & Ownership
-   **Creation:** Instances are not created manually with the `new` keyword. They are instantiated and populated by the server's asset loading pipeline at startup. The static `CODEC` field defines the deserialization logic, mapping data from a configuration file (e.g., JSON) into a Java object. The `afterDecode` hook is used to process the raw string arrays (`fluids`, `weathers`) into high-performance `IntOpenHashSet` collections (`fluidIds`, `weatherIds`) for rapid lookups during gameplay.
-   **Scope:** An instance of this asset persists for the entire lifetime of the server. Once loaded, it is effectively immutable and is shared across the entire server.
-   **Destruction:** The object is garbage collected when the server shuts down and the central asset registry is cleared.

## Internal State & Concurrency
-   **State:** This asset is **effectively immutable** after its initial creation and decoding. Its internal fields, such as `fluidIds` and `weatherIds`, are populated once during the `afterDecode` lifecycle hook and are not modified during gameplay. This design makes the asset inherently safe to share across different systems.
-   **Thread Safety:** The object itself is thread-safe due to its immutable nature. However, its core method, `getCurrentGrowthMultiplier`, is designed to operate exclusively within the server's main update loop and requires a valid `CommandBuffer` context. **WARNING:** Calling this method from an asynchronous task or a different thread without a proper engine context will lead to severe concurrency issues, data corruption, or server crashes. The `CommandBuffer` is the designated mechanism for managing safe access to and mutation of world state.

## API Surface
The public API is minimal, with the core logic encapsulated in a single override. Getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCurrentGrowthMultiplier(...) | double | O(C) | Calculates the growth multiplier based on water sources. Queries adjacent blocks and global weather state. May queue a component update for the underlying TilledSoilBlock via the CommandBuffer. Complexity is constant-time relative to world size, involving a fixed number of neighbor checks and a limited vertical scan for rain obstruction. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most developers. It is configured within a parent asset (e.g., a crop definition file) and is called automatically by the server's farming or block ticking systems. The following example is conceptual, illustrating how an engine system might use an instance of this asset.

```java
// Conceptual example from a hypothetical FarmingSystem
void processCropGrowth(CropComponent crop, CommandBuffer<ChunkStore> cmd) {
    // Assume the crop's asset config has a list of growth modifiers
    for (GrowthModifierAsset modifier : crop.getAsset().getGrowthModifiers()) {
        if (modifier instanceof WaterGrowthModifierAsset) {
            // The system invokes the modifier, passing the necessary world context
            double multiplier = modifier.getCurrentGrowthMultiplier(cmd, ...);
            crop.applyGrowth(BASE_GROWTH_RATE * multiplier);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new WaterGrowthModifierAsset()`. This bypasses the asset loading pipeline and the critical `CODEC` logic, resulting in a non-functional object with null fields. Assets must only be defined in configuration files.
-   **State Mutation:** Do not attempt to modify the `fluidIds` or `weatherIds` collections at runtime. These are treated as immutable after the server starts.
-   **Contextless Invocation:** Do not call `getCurrentGrowthMultiplier` outside of a valid, engine-provided `CommandBuffer` context. The method relies entirely on this context to safely read and write world data.

## Data Pipeline
The flow of data and logic during a single invocation is deterministic and follows a clear path from world state input to a multiplier output.

> Flow:
> Block Tick Event -> Farming System -> **WaterGrowthModifierAsset.getCurrentGrowthMultiplier()**
> 1.  **Query World State:** Reads adjacent block IDs from `FluidSection` components.
> 2.  **Query Global State:** Reads current weather from the global `WeatherResource`.
> 3.  **Internal Logic:** Compares world data against its configured `fluidIds` and `weatherIds`. Performs a vertical scan to check for rain obstruction.
> 4.  **Queue State Change:** Enqueues a component update for the `TilledSoilBlock` on the `CommandBuffer` to set its `hasExternalWater` state.
> 5.  **Return Value:** Returns a `double` multiplier to the calling Farming System.


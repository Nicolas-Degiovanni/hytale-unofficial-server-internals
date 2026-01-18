---
description: Architectural reference for FertilizerGrowthModifierAsset
---

# FertilizerGrowthModifierAsset

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.modifiers
**Type:** Configuration Asset

## Definition
```java
// Signature
public class FertilizerGrowthModifierAsset extends GrowthModifierAsset {
```

## Architecture & Concepts
The FertilizerGrowthModifierAsset is a data-driven component within the server-side farming system. It represents a specific rule for accelerating crop growth. Its primary function is to check the state of the block directly beneath a growing plant and apply a growth multiplier if that block is a fertilized TilledSoilBlock.

This class is a concrete implementation of the abstract GrowthModifierAsset. It embodies the engine's design philosophy of separating game logic from game data. Instead of hard-coding growth rules, game designers can define this modifier in an asset file and associate it with specific plants. The game engine, during its block update cycle, dynamically invokes the logic defined here.

It operates exclusively on the server and interacts with the world state through a CommandBuffer, a core architectural pattern in Hytale for ensuring safe, deferred, and deterministic access to world data like chunks and block components.

### Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically via a constructor. They are deserialized and instantiated by the asset loading system using the provided static CODEC when the server boots and loads its game assets.
-   **Scope:** An instance of this asset is effectively a singleton for its configuration. It is immutable and persists for the entire lifetime of the server session. It is shared and referenced by any other assets (e.g., plant configurations) that use this growth rule.
-   **Destruction:** The object is garbage collected when the server shuts down and its central asset registries are cleared.

## Internal State & Concurrency
-   **State:** This class is stateless and immutable. It contains no instance fields and its behavior is determined entirely by the arguments passed to its methods, primarily the world state accessed via the CommandBuffer.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. The safety of its operations is guaranteed by the CommandBuffer system, which is designed to manage concurrent access to the world state from different systems or worker threads. The getCurrentGrowthMultiplier method can be considered a pure function relative to the world state snapshot provided by the CommandBuffer.

## API Surface
The public contract is minimal, consisting of one overridden method that implements the core logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCurrentGrowthMultiplier(...) | double | O(1) | Returns a growth multiplier. Complexity is constant time but involves several chained lookups into the world state via the CommandBuffer to inspect the block at y-1. |

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class directly in Java code. Instead, it is specified within a data file for another asset, such as a crop. The farming system then retrieves this modifier from the crop's configuration and invokes it during a block update tick.

The conceptual system-level usage would look like this:

```java
// Executed by the server's block update or farming system
void processPlantGrowth(PlantAsset plant, CommandBuffer<ChunkStore> cmd, Ref<ChunkStore> blockRef, int x, int y, int z) {
    double totalMultiplier = 1.0;

    // The system iterates over all configured modifiers for the plant
    for (GrowthModifierAsset modifier : plant.getGrowthModifiers()) {
        // The system invokes the modifier, which could be a FertilizerGrowthModifierAsset
        totalMultiplier *= modifier.getCurrentGrowthMultiplier(cmd, ...);
    }

    // ... apply growth based on totalMultiplier
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new FertilizerGrowthModifierAsset()`. The asset system is responsible for its creation and lifecycle. Direct instantiation bypasses the asset registry and will lead to unmanaged, non-functional objects.
-   **Incorrect Configuration:** Applying this modifier to an asset for a plant that cannot be placed on tilled soil (e.g., a cactus or an aquatic plant) is a logical error. The modifier will always return a multiplier of 1.0, adding useless processing to the block tick.

## Data Pipeline
This component acts as a conditional step in the data pipeline for calculating plant growth.

> Flow:
> Block Update Tick -> Farming System -> Read Plant Asset Config -> **FertilizerGrowthModifierAsset.getCurrentGrowthMultiplier** -> Read World State (Block at Y-1) -> Return Multiplier -> Farming System -> Calculate New Growth State


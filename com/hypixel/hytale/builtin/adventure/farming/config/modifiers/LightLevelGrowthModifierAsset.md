---
description: Architectural reference for LightLevelGrowthModifierAsset
---

# LightLevelGrowthModifierAsset

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.modifiers
**Type:** Data Asset / Configuration

## Definition
```java
// Signature
public class LightLevelGrowthModifierAsset extends GrowthModifierAsset {
```

## Architecture & Concepts
The LightLevelGrowthModifierAsset is a data-driven component that implements a specific rule within the server-side farming system. It acts as a conditional modifier, determining how ambient light levels—both artificial (torches, lamps) and natural (sunlight, skylight)—affect the growth rate of a plant or crop.

Architecturally, this class embodies the **Strategy Pattern**. It is one of many possible implementations of the abstract GrowthModifierAsset. The core farming system is not concerned with the specifics of *why* a growth rate is modified; it simply queries a list of associated modifiers for a given block. This class encapsulates the complex logic of sampling light data from the world, comparing it against configured thresholds, and returning a growth multiplier.

This decoupling allows game designers to define intricate, light-based growth behaviors in simple data files (e.g., JSON) without requiring any changes to the core engine code. The static CODEC field is the primary mechanism for this data-driven approach, defining the schema for deserializing designer-authored configuration into a functional Java object.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually using the *new* keyword. They are deserialized and instantiated by the server's asset loading pipeline at startup, using the public static CODEC. The server reads a configuration file (e.g., a block definition asset) that specifies this modifier and its parameters.

- **Scope:** An instance of LightLevelGrowthModifierAsset is effectively a singleton for its specific configuration. It is loaded once and held in memory by an asset manager for the entire server session. The same instance is referenced by all blocks that share its configuration.

- **Destruction:** The object is garbage collected when the server shuts down or when a full asset reload is triggered, at which point the asset manager releases all references.

## Internal State & Concurrency
- **State:** The internal state of this asset (the light ranges and the requireBoth flag) is **immutable** after its initial creation via the CODEC. All fields are configured during the deserialization process and are not modified during runtime.

- **Thread Safety:** This class is **thread-safe**. The primary method, getCurrentGrowthMultiplier, is a pure function with respect to the object's internal state. It reads from its own immutable fields and from world data provided via the CommandBuffer. The Hytale server architecture is designed to process game logic, such as block ticks, in parallel. This component is safe to use in such an environment because it does not modify its own state or any shared global state.

    **Warning:** Thread safety is contingent on the CommandBuffer providing a consistent and thread-safe view of the world state for the duration of the method call. This is a guarantee provided by the calling system (the block ticking scheduler).

## API Surface
The public contract is focused on a single operation: calculating the current growth multiplier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCurrentGrowthMultiplier(...) | double | O(1) | Calculates the growth multiplier based on the light level at a specific block coordinate. Reads light data from the ChunkStore and time data from the WorldTimeResource. Returns 1.0 if conditions are not met. |

## Integration Patterns

### Standard Usage
This asset is not intended to be invoked directly by gameplay programmers. It is configured as part of a block's properties and is executed automatically by the server's farming or block-ticking systems.

The following conceptual example illustrates how the engine's farming system would use this modifier:

```java
// Hypothetical engine code within a FarmingSystem
void processGrowthTick(Block targetBlock) {
    // 1. The system retrieves all configured modifiers for the block type.
    List<GrowthModifierAsset> modifiers = targetBlock.getType().getGrowthModifiers();
    double finalMultiplier = 1.0;

    // 2. It iterates through them and accumulates their effects.
    for (GrowthModifierAsset modifier : modifiers) {
        // When the modifier is a LightLevelGrowthModifierAsset, its logic is executed here.
        finalMultiplier *= modifier.getCurrentGrowthMultiplier(commandBuffer, ...);
    }

    // 3. The final multiplier is applied to the block's growth state.
    targetBlock.applyGrowth(finalMultiplier);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new LightLevelGrowthModifierAsset()`. This bypasses the asset system and results in an unconfigured, non-functional object. All instances must be loaded from asset files.

- **State Mutation:** Do not attempt to modify the internal fields (e.g., `sunlight`, `artificialLight`) after the asset has been loaded. These are intended to be immutable configuration data. Modifying them at runtime can lead to unpredictable behavior across the server.

## Data Pipeline
The flow of data for a single growth calculation is linear and synchronous, triggered by a server game tick.

> Flow:
> Server Tick Scheduler -> Farming System identifies a block for update -> **LightLevelGrowthModifierAsset.getCurrentGrowthMultiplier** is invoked -> Reads `ChunkLightData` and `WorldTimeResource` -> Returns a `double` multiplier -> Farming System applies multiplier to block's growth component.


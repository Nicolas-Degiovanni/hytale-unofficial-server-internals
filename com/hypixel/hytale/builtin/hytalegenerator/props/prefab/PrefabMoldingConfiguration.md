---
description: Architectural reference for PrefabMoldingConfiguration
---

# PrefabMoldingConfiguration

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.prefab
**Type:** Transient

## Definition
```java
// Signature
public class PrefabMoldingConfiguration {
```

## Architecture & Concepts
PrefabMoldingConfiguration is an immutable data-holding object that encapsulates all the parameters required for a world generator's "molding" pass. In the context of procedural generation, molding is the process of shaping the terrain and environment immediately surrounding a newly placed prefab to ensure it blends naturally into the world.

This class acts as a parameter object, decoupling the prefab placement logic from the specific terrain modification algorithms. Instead of passing numerous individual arguments (a scanner, a pattern, a direction), the generator passes a single, self-contained PrefabMoldingConfiguration instance. This improves code clarity and simplifies the extension of molding capabilities, as new parameters can be added to this configuration object without changing the method signatures of the core placement engine.

Its primary consumer is the world generation pipeline, which reads the configuration after placing a prefab and executes the specified terrain alteration task.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the world generation system when it processes a prefab that includes molding properties. It is not managed by a dependency injection container or service registry.
- **Scope:** Extremely short-lived. An instance typically exists only for the duration of a single prefab placement operation. It is created, passed to the molding algorithm, and then becomes eligible for garbage collection.
- **Destruction:** Cleaned up by the Java garbage collector once it is no longer referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** Strictly immutable. All member fields are declared `public final` and are assigned exclusively within the constructor. Once an instance is created, its state cannot be altered.
- **Thread Safety:** Inherently thread-safe due to its immutability. An instance can be safely shared and read across multiple world generation threads without any need for locks or synchronization. This is a critical design feature for a highly parallelized procedural generation engine.

## API Surface
The public contract consists of the constructor and a static factory method. The public fields are intended for direct, read-only access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| none() | PrefabMoldingConfiguration | O(1) | Static factory method that returns a no-op configuration. This implements the Null Object Pattern, allowing generator code to handle prefabs without molding properties gracefully without requiring explicit null checks. |

## Integration Patterns

### Standard Usage
The configuration is created directly with the required molding parameters and passed into the generation pipeline.

```java
// Define the molding parameters for a specific prefab
Scanner terrainScanner = new TerrainHeightScanner(...);
Pattern grassPattern = new BlockPattern(Block.GRASS);

// Create the configuration object
PrefabMoldingConfiguration moldingConfig = new PrefabMoldingConfiguration(
    terrainScanner,
    grassPattern,
    MoldingDirection.DOWNWARDS,
    true
);

// Pass the configuration to the world generator during placement
worldGenerator.placePrefab(somePrefab, position, moldingConfig);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify an instance after creation. Its immutable design forbids this. If different parameters are needed, a new instance must be created.
- **Null Parameters:** While the `none()` factory method creates a valid no-op configuration, avoid passing `null` directly to the constructor. This can lead to NullPointerExceptions in the downstream molding algorithms which expect a valid, non-null configuration object (even if it is a no-op configuration from `none()`).

## Data Pipeline
This class does not process data itself; it *is* the data that configures a stage in the world generation pipeline.

> Flow:
> Prefab Asset Definition -> **PrefabMoldingConfiguration (Instantiation)** -> World Generator Placement Task -> Terrain Molding Algorithm -> World Voxel State Modification


---
description: Architectural reference for ClimateColor
---

# ClimateColor

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Value Object

## Definition
```java
// Signature
public class ClimateColor {
```

## Architecture & Concepts
The ClimateColor class is an immutable data container that encapsulates the four primary color values used to visually represent a climate or biome on a world map. It serves as a fundamental data structure within the server-side procedural world generation engine.

Architecturally, this class functions as a Value Object. Its identity is defined entirely by the data it holds—the `land`, `shore`, `ocean`, and `shallowOcean` colors. By enforcing immutability through `final` fields, it guarantees that a climate's color palette is stable and cannot be mutated at runtime. This is critical for achieving deterministic and reproducible world generation across different threads and server instances.

The static constants, such as `ISLAND` and `OCEAN`, provide globally accessible, pre-defined integer color values for common or default geographical features, separate from the instantiable color palettes.

## Lifecycle & Ownership
- **Creation:** ClimateColor instances are typically created during the server's bootstrap phase by configuration loaders. Systems responsible for parsing biome or climate definition files (e.g., from JSON or HOCON) will instantiate this class to populate the color data for each defined biome. Direct instantiation in game logic is rare and usually reserved for programmatic biome definition.
- **Scope:** The lifetime of a ClimateColor object is bound to its owning `Biome` or `Climate` definition object. It is not a session-scoped object but rather a configuration-scoped one. It persists in memory as long as its parent biome definition is loaded.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once the `Biome` or `Climate` definition they belong to is unloaded and no longer referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The class is **deeply immutable**. Its state consists of four primitive `int` fields, all of which are declared `final` and are set only once during construction. The object's state cannot be changed after it has been created.
- **Thread Safety:** ClimateColor is **inherently thread-safe**. Due to its immutability, an instance can be safely shared and read by multiple world generation worker threads simultaneously without any risk of data corruption or race conditions. No synchronization mechanisms, such as locks, are required.

## API Surface
The public contract is defined by its constructor and direct field access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ClimateColor(land, shore, ocean, shallowOcean) | Constructor | O(1) | Constructs a new, immutable climate color palette. |
| land, shore, ocean, shallowOcean | public final int | O(1) | Provides direct, read-only access to the integer color values. |
| UNSET, ISLAND, OCEAN, etc. | public static final int | O(1) | Provides access to pre-defined, globally shared color constants. |

## Integration Patterns

### Standard Usage
A ClimateColor object is almost always retrieved from a higher-level data structure, such as a Biome definition. It is then used by rendering or map generation systems to determine the correct pixel color for a given terrain type.

```java
// Correctly retrieve the color palette from a biome definition
Biome currentBiome = world.getBiomeAt(x, z);
ClimateColor palette = currentBiome.getClimateColor();

// Use the palette to determine the final map color
int mapPixelColor = palette.land;
```

### Anti-Patterns (Do NOT do this)
- **Attempted Modification:** Do not attempt to design systems that rely on modifying a ClimateColor object after its creation. Its immutability is a core design principle. If a different palette is needed, a new ClimateColor instance must be created.
- **Misusing Constants:** The static integer constants like `OCEAN` are not complete palettes. Do not pass these raw integer values to methods expecting a ClimateColor object. This will result in a type mismatch and compilation failure.

## Data Pipeline
ClimateColor acts as a data source, not a processing stage. It represents configuration data that has been loaded into memory for use by the world generator.

> Flow:
> Biome Definition File (e.g., JSON) -> Configuration Parser -> **ClimateColor Instance** -> Biome Definition Object -> World Generation Algorithm -> Map Color Buffer


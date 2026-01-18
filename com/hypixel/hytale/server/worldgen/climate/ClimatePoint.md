---
description: Architectural reference for ClimatePoint
---

# ClimatePoint

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ClimatePoint {
```

## Architecture & Concepts
The ClimatePoint class is a fundamental, high-performance data structure used exclusively within the server-side world generation pipeline. It is not a service or manager; rather, it is a simple data carrier, or Plain Old Java Object (POJO), that represents the complete set of climatic conditions at a specific coordinate in the world.

Its primary role is to hold the intermediate state of climate calculations. World generation begins by sampling base noise maps (e.g., for temperature and humidity), which are used to instantiate a ClimatePoint. This object is then passed through a chain of modifiers and filters that mutate its state directly, layering effects such as altitude, proximity to oceans, or magical influences. The final, mutated state of the ClimatePoint is then used by a BiomeSelector to determine the definitive biome for that coordinate.

The design, featuring public mutable fields, prioritizes raw performance and low memory allocation overhead, which is critical for the computationally expensive task of generating vast worlds.

### Lifecycle & Ownership
-   **Creation:** A ClimatePoint is instantiated on-the-fly by climate sampling services deep within the world generation process. A new instance is typically created for each unique coordinate (x, z) being evaluated.
-   **Scope:** The object's lifetime is extremely brief and ephemeral. It exists only within the stack frame of the generation method processing a single point or a small region. It is not intended to be stored, cached, or referenced long-term.
-   **Destruction:** The object is abandoned after the final biome is determined and becomes immediately eligible for garbage collection. Its memory footprint is reclaimed aggressively by the JVM.

## Internal State & Concurrency
-   **State:** The state is comprised of three public `double` fields: temperature, intensity, and modifier. This class is intentionally and fully **Mutable**. Any component with a reference to a ClimatePoint instance can directly read or write to its fields. This design avoids the overhead of getters, setters, and defensive copying.

-   **Thread Safety:** This class is **Not Thread-Safe**. Its public, mutable fields make it highly susceptible to race conditions and data corruption if a single instance is accessed concurrently by multiple threads.

    **WARNING:** Never share a ClimatePoint instance between different world generation worker threads. Each thread must create and operate on its own distinct instances to ensure deterministic and correct world generation.

## API Surface
The public contract of ClimatePoint consists of its constructors and direct field access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ClimatePoint(temp, intensity) | Constructor | O(1) | Creates a new point with a default modifier of 1.0. |
| ClimatePoint(temp, intensity, mod) | Constructor | O(1) | Creates a new point with all three climate values specified. |
| temperature | double | O(1) | Direct access to the temperature value. |
| intensity | double | O(1) | Direct access to the intensity value (often representing humidity or rainfall). |
| modifier | double | O(1) | A generic modifier for biome-specific calculations. |
| EMPTY_ARRAY | ClimatePoint[] | O(1) | A static, shared instance of an empty array to prevent unnecessary allocations. |

## Integration Patterns

### Standard Usage
A ClimatePoint is created by a sampler and then passed to a series of services that mutate it in-place before a final decision is made.

```java
// Conceptual example within a world generator
ClimateSampler sampler = context.getService(ClimateSampler.class);
BiomeModifier altitudeModifier = context.getService(AltitudeModifier.class);
BiomeSelector selector = context.getService(BiomeSelector.class);

// 1. A new, ephemeral point is created for the coordinate
ClimatePoint point = sampler.getClimateAt(x, z);

// 2. The point is passed to a modifier which mutates its fields directly
altitudeModifier.apply(point, worldHeight);

// 3. The final state is used to select a biome
Biome finalBiome = selector.selectBiomeFrom(point);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store ClimatePoint instances in caches, member variables, or any persistent data structure. They are designed to be short-lived.
-   **Inter-Thread Sharing:** Do not pass a ClimatePoint from one worker thread to another. This will lead to unpredictable world generation bugs that are difficult to reproduce.
-   **Assuming Immutability:** Do not pass a ClimatePoint to a method and expect its values to remain unchanged. The explicit design contract is that downstream methods are allowed, and expected, to mutate the object.

## Data Pipeline
The ClimatePoint acts as the payload that flows through the climate calculation stage of the world generation pipeline.

> Flow:
> Noise Maps (Temperature, Humidity) -> ClimateSampler -> **new ClimatePoint** -> Biome Modifiers (mutate instance) -> BiomeSelector (read instance) -> Final Biome ID


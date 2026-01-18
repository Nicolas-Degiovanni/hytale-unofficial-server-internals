---
description: Architectural reference for NoiseMaskCondition
---

# NoiseMaskCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public class NoiseMaskCondition implements ICoordinateCondition {
```

## Architecture & Concepts
The NoiseMaskCondition is a fundamental component in the procedural generation pipeline. It functions as a high-level **Adapter** that bridges two distinct concepts: continuous value fields (noise) and discrete boolean checks (conditions).

Its primary architectural role is to translate a floating-point value from a NoiseProperty at a specific world coordinate into a simple true or false result. It achieves this through composition, wrapping a NoiseProperty and an IDoubleCondition.

This class enables developers to define complex spatial regions based on procedural noise. For example, it can be used to answer questions like, "Is the terrain elevation noise at this coordinate above 0.7?" or "Is the humidity value within the range of 0.2 to 0.5?". It is a core building block for defining biome boundaries, ore vein distribution, and cave system placement.

### Lifecycle & Ownership
- **Creation:** NoiseMaskCondition is instantiated dynamically by a higher-level procedural generation system, typically when a world generation "recipe" or "profile" is being constructed. It is not managed by a dependency injection container and is created via its public constructor.
- **Scope:** The object's lifetime is ephemeral and is strictly tied to the scope of the generation task that created it. It typically exists only for the duration of a single chunk or region generation pass.
- **Destruction:** The object is managed by the Java Garbage Collector. Once the generation pass completes and all references are dropped, it is eligible for collection. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields for the NoiseProperty and IDoubleCondition are marked as final and are injected during construction. The object holds no mutable state, and its evaluation methods produce no side effects.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable and stateless design, a single instance of NoiseMaskCondition can be safely shared and executed by multiple world generation threads concurrently without locks or synchronization. This is a critical design feature for achieving high-performance, parallelized world generation.

    **Warning:** While the NoiseMaskCondition itself is thread-safe, this guarantee assumes that the composed NoiseProperty and IDoubleCondition objects are also thread-safe.

## API Surface
The public contract is exclusively defined by the ICoordinateCondition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(seed, x, y) | boolean | O(N) | Evaluates the 2D noise mask. Returns true if the noise value at (x, y) satisfies the internal condition. |
| eval(seed, x, y, z) | boolean | O(N) | Evaluates the 3D noise mask. Returns true if the noise value at (x, y, z) satisfies the internal condition. |

*Note: Complexity is O(N), where N is the computational complexity of the underlying NoiseProperty function (e.g., multi-octave Perlin noise). The overhead of the NoiseMaskCondition wrapper itself is O(1).*

## Integration Patterns

### Standard Usage
This component is designed to be composed within a larger generation algorithm. It is rarely used in isolation.

```java
// 1. Define the noise function to be used as a mask
NoiseProperty elevationNoise = new NoiseProperty(/*...config...*/);

// 2. Define the condition to apply to the noise value
// This condition checks if a value is greater than 0.5
IDoubleCondition threshold = (val) -> val > 0.5;

// 3. Create the NoiseMaskCondition
ICoordinateCondition isHighAltitude = new NoiseMaskCondition(elevationNoise, threshold);

// 4. Use the condition in a generation loop
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        if (isHighAltitude.eval(worldSeed, worldX + x, 0, worldZ + z)) {
            // Place mountain block
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Re-instantiation in Loops:** Do not create a new NoiseMaskCondition inside a tight generation loop (e.g., per-block). This is highly inefficient and will cause significant GC pressure. Instantiate it once per generation task and reuse it for all coordinates.
- **Stateful Conditions:** Do not inject an IDoubleCondition that relies on mutable external state. This breaks the thread-safety guarantee and will lead to unpredictable, non-deterministic generation results in a multithreaded environment.

## Data Pipeline
The data flow through this component is a direct, single-pass transformation. It takes coordinate data as input and produces a boolean as output.

> Flow:
> (seed, x, y, z) -> **NoiseMaskCondition**.noiseMask -> double value -> **NoiseMaskCondition**.condition -> boolean result


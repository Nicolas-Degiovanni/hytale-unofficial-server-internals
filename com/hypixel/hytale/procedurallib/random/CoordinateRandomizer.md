---
description: Architectural reference for CoordinateRandomizer
---

# CoordinateRandomizer

**Package:** com.hypixel.hytale.procedurallib.random
**Type:** Transient

## Definition
```java
// Signature
public class CoordinateRandomizer implements ICoordinateRandomizer {
```

## Architecture & Concepts

The CoordinateRandomizer is a foundational component within the procedural generation library. Its primary function is to apply deterministic, noise-based displacement to a given 2D or 3D coordinate. This system is critical for breaking up grid-like patterns and introducing organic, natural-looking variation into the world.

Architecturally, it acts as a stateless computational engine. It is configured with a set of noise functions, each associated with a specific amplitude, for every axis (X, Y, Z). When a coordinate is passed in, the randomizer evaluates each noise function for the relevant axes at that coordinate, scales the result by its amplitude, and sums the offsets to produce a new, displaced coordinate.

The determinism is key: for a given seed, input coordinate, and configuration, the output will always be identical. This ensures that procedurally generated worlds are reproducible. The use of multiple noise functions per axis allows for the composition of complex displacement patterns, such as combining low-frequency noise for large-scale shifts with high-frequency noise for fine-grained surface detail.

The class also provides a null object implementation, EMPTY_RANDOMIZER, which performs no operation and returns the original coordinate. This is useful as a default or for disabling displacement effects without null checks.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor. It is typically created and configured by higher-level procedural systems, such as a biome or feature generator, often based on definitions loaded from external data files.
- **Scope:** The lifetime of a CoordinateRandomizer instance is bound to its owner. It is a configuration-driven object, not a global service, and persists only as long as the system that requires its specific displacement logic.
- **Destruction:** Managed by the Java garbage collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state consists of three final arrays: xNoise, yNoise, and zNoise. These arrays hold AmplitudeNoiseProperty objects, which encapsulate the noise function and its corresponding strength. This state is provided at construction and is not intended to be modified thereafter. The object is effectively immutable.
- **Thread Safety:** This class is thread-safe. Its methods are pure functions that only read from the immutable construction-time state. The core computation is delegated to NoiseProperty instances. Assuming the underlying NoiseProperty implementations are themselves thread-safe (a standard requirement for noise libraries), multiple threads can safely call the randomization methods on a shared CoordinateRandomizer instance without synchronization. This is essential for parallelized world generation chunks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| randomDoubleX(seed, x, y) | double | O(N) | Calculates the displaced X coordinate in a 2D context. N is the number of noise functions for the X axis. |
| randomDoubleY(seed, x, y) | double | O(N) | Calculates the displaced Y coordinate in a 2D context. N is the number of noise functions for the Y axis. |
| randomDoubleX(seed, x, y, z) | double | O(N) | Calculates the displaced X coordinate in a 3D context. N is the number of noise functions for the X axis. |
| randomDoubleY(seed, x, y, z) | double | O(N) | Calculates the displaced Y coordinate in a 3D context. N is the number of noise functions for the Y axis. |
| randomDoubleZ(seed, x, y, z) | double | O(N) | Calculates the displaced Z coordinate in a 3D context. N is the number of noise functions for the Z axis. |

## Integration Patterns

### Standard Usage
A CoordinateRandomizer is configured during the initialization of a procedural system. It is then used repeatedly to displace coordinates during the generation process, for example, to determine the final placement of trees or rocks.

```java
// Assume noise1 and noise2 are pre-configured NoiseProperty instances
CoordinateRandomizer.AmplitudeNoiseProperty[] xDisplacement = {
    new CoordinateRandomizer.AmplitudeNoiseProperty(noise1, 5.0), // Large scale displacement
    new CoordinateRandomizer.AmplitudeNoiseProperty(noise2, 0.5)  // Fine detail jitter
};

// Y and Z are not displaced in this example
CoordinateRandomizer.AmplitudeNoiseProperty[] empty = {};

CoordinateRandomizer randomizer = new CoordinateRandomizer(xDisplacement, empty, empty);

// Displace a point during world generation
double originalX = 100.0;
double originalY = 50.0;
int worldSeed = 12345;

double finalX = randomizer.randomDoubleX(worldSeed, originalX, originalY);
```

### Anti-Patterns (Do NOT do this)
- **Post-Construction State Mutation:** Although the inner class AmplitudeNoiseProperty has setters, modifying its state after the CoordinateRandomizer has been constructed and potentially shared is a severe anti-pattern. It breaks the assumption of immutability and can lead to non-deterministic or unpredictable behavior in a multi-threaded environment.
- **Excessive Noise Functions:** While powerful, using a large number of noise functions for a single axis will directly impact performance, as each call to a randomDouble method iterates through the entire array. Profile carefully and use the minimum number of functions required to achieve the desired visual effect.

## Data Pipeline
The CoordinateRandomizer is a computational transform component. It does not manage a persistent data stream but rather transforms input data on demand.

> Flow:
> Input Coordinate & Seed -> **CoordinateRandomizer.randomDoubleX()** -> Loop over `xNoise` array -> For each element, call `NoiseProperty.get()` -> Scale result by `amplitude` -> Sum results into `offsetX` -> Return `x + offsetX`


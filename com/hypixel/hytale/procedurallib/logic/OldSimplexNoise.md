---
description: Architectural reference for OldSimplexNoise
---

# OldSimplexNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Singleton

## Definition
```java
// Signature
public class OldSimplexNoise implements NoiseFunction {
```

## Architecture & Concepts
OldSimplexNoise is a foundational utility within the procedural generation library. It provides a concrete implementation of the NoiseFunction interface, generating 2D and 3D gradient noise based on the Simplex Noise algorithm.

Architecturally, this class serves as a pure, deterministic function provider. It is a low-level component with no dependencies on other engine systems, designed to be invoked by higher-level world generation services such as terrain generators, biome placement systems, or procedural texture synthesizers. Its primary role is to transform a set of input coordinates and a seed into a predictable, pseudo-random floating-point value. This determinism is critical for ensuring that worlds can be regenerated identically from the same seed.

The implementation is self-contained and relies on a set of pre-defined mathematical constants and gradient vectors to perform its calculations.

### Lifecycle & Ownership
- **Creation:** The single instance, INSTANCE, is instantiated statically during class loading by the Java Virtual Machine. This occurs the first time any code references the OldSimplexNoise class.
- **Scope:** The singleton instance persists for the entire lifetime of the application. It is globally accessible via the public static final field.
- **Destruction:** The object is not explicitly destroyed. It is eligible for garbage collection only when the application's ClassLoader is unloaded, which typically happens at JVM shutdown.

## Internal State & Concurrency
- **State:** This class is stateless from an object instance perspective. It contains no instance fields. All data, such as gradient vectors and mathematical constants, are stored in static final fields. This data is initialized once at class-load time and is immutable thereafter.
- **Thread Safety:** **Fully Thread-Safe.** The get methods are pure functions; their output depends solely on their input arguments. They do not modify any shared state. Consequently, the public static INSTANCE can be safely shared and invoked by multiple threads concurrently without external synchronization. This is a crucial feature for enabling parallelized chunk generation.

## API Surface
The public contract is defined by the NoiseFunction interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Computes the 2D Simplex noise value for the given coordinates and seed. The complexity is constant time but involves a significant number of floating-point operations. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Computes the 3D Simplex noise value for the given coordinates and seed. The complexity is constant time but computationally more expensive than the 2D variant. |

## Integration Patterns

### Standard Usage
Always access the shared singleton instance. Do not attempt to create a new object. This instance is typically passed to or used by systems that require procedural noise.

```java
// Correctly obtain the singleton instance
NoiseFunction noise = OldSimplexNoise.INSTANCE;

// Use it to generate a value for a specific world coordinate
int worldSeed = 12345;
double heightValue = noise.get(worldSeed, 0, 150.5, -88.2);

// Use the value in a higher-level system
if (heightValue > 0.5) {
    // place stone block
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Any attempt to instantiate this class using reflection will break the design contract and is unsupported.
- **Misuse for Cryptography:** Simplex noise is deterministic and its patterns can be analyzed. It is **not** a cryptographically secure random number generator and must not be used for security-sensitive purposes.
- **Ignoring the Seed:** Using a constant seed (e.g., always 0) will produce the exact same noise field every time. The seed is the primary mechanism for creating unique procedural content.

## Data Pipeline
OldSimplexNoise acts as a data source or generator in a procedural pipeline. It does not transform streams of data but rather creates a single data point on demand.

> Flow:
> (Integer Seed, Double Coordinates) -> **OldSimplexNoise.get()** -> Double Noise Value [-1.0, 1.0] -> Terrain Heightmap / Biome Map / etc.


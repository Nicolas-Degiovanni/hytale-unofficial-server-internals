---
description: Architectural reference for HashUtil
---

# HashUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public class HashUtil {
```

## Architecture & Concepts
HashUtil is a core, stateless utility class providing high-performance, deterministic hashing functions. It serves as a fundamental building block for any system requiring repeatable pseudo-random number generation or fast object hashing, most notably procedural content generation (PCG).

The internal algorithm is a non-cryptographic integer hash function, optimized for speed and a good avalanche effect—small changes to input values result in significant, uncorrelated changes to the output hash. This property is essential for generating varied yet predictable outcomes from a set of input seeds, such as world coordinates or entity identifiers.

Its primary role within the engine is to transform a set of seed values (typically coordinates and a global world seed) into a stable numerical foundation for more complex algorithms, like noise generation (Perlin, Simplex) or loot table selection. By centralizing this logic, the engine ensures that the entire game world and its behaviors can be deterministically reproduced from a single initial seed.

**WARNING:** This class provides non-cryptographic hashing. It is not designed for security-sensitive applications. Do not use these functions for password storage, data integrity verification, or authentication.

## Lifecycle & Ownership
- **Creation:** The HashUtil class is never instantiated. Its constructor is private to enforce a purely static access pattern. The class is loaded into the JVM by the ClassLoader when first referenced.
- **Scope:** As a static utility, its methods are globally accessible throughout the application's lifetime. It holds no state and is not tied to any specific game session or world instance.
- **Destruction:** The class is unloaded from the JVM upon application shutdown. No manual cleanup is ever required.

## Internal State & Concurrency
- **State:** HashUtil is completely **stateless**. All methods are pure functions, meaning their output is determined exclusively by their input arguments. The class contains no member variables, caches, or any other form of internal state.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless and purely functional nature, all methods can be invoked concurrently from any number of threads without risk of race conditions or data corruption. No external locking or synchronization is necessary.

## API Surface
The public API consists of static methods for hashing and pseudo-random number generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hash(long...) | long | O(1) | Computes a deterministic hash from one to four long integer inputs. |
| rehash(long...) | long | O(1) | Applies the hash function twice to the inputs for improved bit distribution. |
| random(long...) | double | O(1) | Generates a deterministic pseudo-random double between 0.0 and 1.0 from inputs. |
| randomInt(long, long, long, int) | int | O(1) | Generates a deterministic pseudo-random integer within a specified bound. |
| hashUuid(UUID) | long | O(1) | Computes a stable hash from a standard Java UUID object. |

## Integration Patterns

### Standard Usage
HashUtil should be invoked statically wherever a deterministic numerical result is needed from a set of seed values. It is commonly used as the first step in procedural generation chains.

```java
// Example: Generating a consistent height value for a world column
long worldSeed = 123456789L;
int blockX = 150;
int blockZ = -45;

// Generate a stable random value for this specific coordinate
double heightNoise = HashUtil.random(worldSeed, blockX, blockZ);

// Use the noise value to determine terrain height
int terrainHeight = 64 + (int)(heightNoise * 32);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance of this class. The private constructor will prevent this, and it violates the intended static-only design.
- **Security-Sensitive Hashing:** Do not use HashUtil to hash passwords or verify file integrity. The algorithm is not collision-resistant against malicious attacks. Use a standard cryptographic library like JCE's MessageDigest for such tasks.
- **Unpredictable Randomness:** Do not use HashUtil for features requiring true unpredictability, such as generating encryption keys or for online gambling mechanics. Use java.security.SecureRandom for cryptographically strong random numbers.

## Data Pipeline
HashUtil acts as a pure transformation step within a larger data processing pipeline. It does not originate or terminate a flow but is a critical computational kernel.

> **Flow for Procedural Terrain:**
> World Seed & Block Coordinates -> **HashUtil.random()** -> Noise Algorithm (e.g., Perlin) -> Biome-specific Modifiers -> Final Voxel Type


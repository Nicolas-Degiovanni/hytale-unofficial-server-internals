---
description: Architectural reference for SeedString
---

# SeedString

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class SeedString<T extends SeedResource> {
```

## Architecture & Concepts
The SeedString class is an immutable value object that represents a hierarchical seed used within the procedural generation engine. It is a foundational component for ensuring deterministic and reproducible world generation.

Its core design principle is the fusion of a string-based seed with a strongly-typed resource context, represented by the generic parameter `T`. This prevents the system from treating seeds as simple strings, instead enforcing that they carry associated metadata and resources required for generation.

The class distinguishes between an `original` string and a derived `seed` string. This allows algorithms to create variations and specializations of a seed while preserving the identity of its origin. This pattern is critical for layered procedural generation, where a base biome seed might be appended with identifiers for specific features like trees, rocks, or dungeons.

Because SeedString is immutable, every operation that appears to modify an instance, such as `append`, actually produces a new SeedString object. This design guarantees thread safety and eliminates a significant class of bugs related to state mutation in a highly parallelized engine.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by procedural generation algorithms, world initializers, or configuration parsers. They are not managed by a dependency injection container or service locator. A typical pattern involves creating a root SeedString for a large area (e.g., a world or zone) and then deriving child instances for smaller features.
- **Scope:** Short-lived and function-scoped. A SeedString typically exists for the duration of a specific generation task, passed down through the call stack as a parameter, and is eligible for garbage collection once the task is complete.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no explicit `close` or `destroy` methods, as the object holds no unmanaged resources.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (`original`, `seed`, `t`, `hash`) are final and are assigned only once during construction. The hash code is pre-calculated and cached at instantiation to optimize performance for subsequent use in hash-based collections.
- **Thread Safety:** **Unconditionally thread-safe**. Due to its immutable nature, a SeedString instance can be safely shared, passed, and read across multiple threads without any external synchronization or locks. This is a critical feature for the performance of the parallel procedural generation pipeline.

## API Surface
The public API is designed for fluent, chainable derivation of new seeds.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| append(String suffix) | SeedString | O(N) | Creates a new instance by appending a suffix to the current *seed*. The *original* string is preserved from the parent. |
| appendToOriginal(String suffix) | SeedString | O(N) | Creates a new instance where the new *seed* is the parent's *original* string plus the suffix. |
| alternateOriginal(String suffix) | SeedString | O(N) | Creates a new instance with a modified *original* and a correspondingly updated *seed*. Used for complex path-like transformations. |
| get() | T | O(1) | Retrieves the associated SeedResource context object. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a base seed and progressively specialize it for finer-grained procedural tasks. The return value of derivation methods must always be captured.

```java
// A generator receives a base seed for a forest biome
SeedResource forestResource = context.getResource("hytale:forest_biome");
SeedString<SeedResource> biomeSeed = new SeedString<>("world1.zoneA.forest", forestResource);

// It then derives a more specific seed for a cluster of oak trees
SeedString<SeedResource> featureSeed = biomeSeed.append(".trees.oak_cluster_4");

// This final, specific seed is used to initialize a random number generator
Random random = new Random(featureSeed.hashCode());
generator.placeTrees(random, featureSeed.get());
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Return Values:** The most common error is treating SeedString as mutable. The following code has no effect, as the new instance is discarded.
  > `// WRONG`
  > `baseSeed.append(".feature");`
  > `// CORRECT`
  > `SeedString derivedSeed = baseSeed.append(".feature");`
- **Null Resource Injection:** The constructor will throw a NullPointerException if the SeedResource is null. If no specific resource is applicable, the provided static default must be used.
  > `// WRONG`
  > `new SeedString<>("my.seed", null);`
  > `// CORRECT`
  > `new SeedString<>("my.seed", SeedString.DEFAULT_RESOURCE);`
- **High-Frequency Allocation:** Avoid creating new SeedString instances inside tight, performance-critical loops. While efficient, the object and string allocations can add up. For scenarios requiring extreme performance, consider a specialized mutable builder pattern if available.

## Data Pipeline
SeedString acts as a data carrier that flows through the procedural generation system. It does not process data itself but provides the necessary context and identity for downstream processors.

> Flow:
> World Configuration -> Creates root **SeedString** -> Biome Placement Algorithm (derives biome-level **SeedString**) -> Feature Spawner (derives feature-level **SeedString**) -> Final Generator (consumes the **SeedString** to seed a PRNG and retrieve its resource)


---
description: Architectural reference for SeedBox
---

# SeedBox

**Package:** com.hypixel.hytale.builtin.hytalegenerator.seed
**Type:** Transient

## Definition
```java
// Signature
public class SeedBox {
```

## Architecture & Concepts
The SeedBox is a foundational component for the procedural generation system. It provides a mechanism for creating reproducible, deterministic sequences of random numbers in a hierarchical and isolated manner. Its primary architectural role is to act as a factory for random number suppliers, ensuring that different parts of the world generator can operate independently without corrupting a shared random state.

The core principle is **deterministic derivation**. A root SeedBox is created from a primary world seed. Sub-systems, such as the biome generator or structure placer, can then create their own "child" SeedBox instances. This process can be nested indefinitely. Because the derivation relies on string concatenation and the consistent output of `String.hashCode()`, the same sequence of `child` calls with the same keys will always result in a SeedBox that produces the exact same stream of random numbers.

This design elegantly solves the problem of state management for randomness in a complex, multi-stage generation pipeline. It obviates the need to pass mutable `java.util.Random` objects through the system, which is a common source of bugs and non-deterministic behavior.

### Lifecycle & Ownership
- **Creation:** A root SeedBox is instantiated at the beginning of a world generation task, typically using the global world seed. All subsequent SeedBox instances are created by calling the `child` method on an existing instance.
- **Scope:** SeedBox objects are transient and short-lived. Their lifecycle is typically bound to the scope of the specific generation task that requires a derived random stream. They are value objects passed down the call stack, not long-term stateful services.
- **Destruction:** Instances are managed by the Java garbage collector and are reclaimed once they are no longer referenced. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** Immutable. The internal `key` is a final String, set at construction. Methods like `child` do not modify the instance; they return a new SeedBox instance with a new derived key. This immutability makes the class predictable and safe to pass between components.
- **Thread Safety:** The SeedBox class itself is inherently thread-safe due to its immutability. Multiple threads can safely call `child` or `createSupplier` on the same SeedBox instance concurrently.

   **WARNING:** The `Supplier<Integer>` returned by `createSupplier` is **NOT** thread-safe. It encapsulates a `java.util.Random` instance, which is not designed for concurrent access. Each thread requiring a deterministic random stream must create its own `Supplier` from a SeedBox. Sharing a single supplier across threads will lead to race conditions and break determinism.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| child(String childKey) | SeedBox | O(N) | Creates and returns a new, derived SeedBox. Complexity is proportional to the combined length of the parent and child keys due to string concatenation. |
| createSupplier() | Supplier<Integer> | O(1) | Creates a supplier that provides a stream of pseudo-random integers. The stream is seeded deterministically from the SeedBox's internal key. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a root seed and progressively derive more specific seeds as the generation process drills down into finer details.

```java
// 1. A root SeedBox is created from the global world seed.
SeedBox worldSeed = new SeedBox("hytale-world-42");

// 2. The terrain system derives its own master seed.
SeedBox terrainSeed = worldSeed.child("terrain_generator");

// 3. A specific biome generator derives a seed from the terrain system's seed.
SeedBox forestBiomeSeed = terrainSeed.child("temperate_forest");

// 4. The forest generator creates a supplier to control tree placement.
Supplier<Integer> treePlacementRandom = forestBiomeSeed.createSupplier();
int x = treePlacementRandom.get();
int z = treePlacementRandom.get();
```

### Anti-Patterns (Do NOT do this)
- **Supplier Sharing:** Never create a single `Supplier` and pass it to multiple, independent systems or threads. This couples their random state and will produce incorrect, non-deterministic results. Each system should derive its own SeedBox and create its own Supplier.
- **Direct Instantiation for Derivation:** Do not use `new SeedBox(parentKey + childKey)`. Always use the `child` method to ensure the derivation pattern is clear and consistent throughout the codebase.

## Data Pipeline
The SeedBox facilitates a control flow for deterministic data generation rather than a traditional data processing pipeline.

> Flow:
> Global World Seed (String) -> **Root SeedBox** -> `child()` Derivation -> **Sub-System SeedBox** -> `createSupplier()` -> `Supplier<Integer>` -> Procedural Generation Algorithm


---
description: Architectural reference for MaskShader
---

# MaskShader

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.shaders
**Type:** Transient

## Definition
```java
// Signature
public class MaskShader<T> implements Shader<T> {
```

## Architecture & Concepts
The MaskShader is a foundational component within the procedural generation framework, implementing the **Decorator** design pattern. It acts as a conditional wrapper around another Shader instance, referred to as the *child shader*.

Its primary function is to selectively apply the logic of its child shader based on a provided Predicate, the *mask*. When the `shade` method is invoked, the MaskShader first evaluates the input data against its mask.

-   If the mask Predicate returns **true**, the MaskShader delegates the operation to its child shader, effectively applying the child's transformation.
-   If the mask Predicate returns **false**, the MaskShader bypasses the child shader entirely and returns the input data unmodified.

This architecture enables the composition of complex, layered generation logic. For example, a MaskShader can be used to ensure a "CaveCarver" shader only operates on blocks below a certain altitude, or that a "Forest" shader only applies to biomes with sufficient rainfall. It is a key building block for creating rule-based, conditional transformations in the world generation pipeline.

### Lifecycle & Ownership
-   **Creation:** A MaskShader is never instantiated directly. Its construction is managed exclusively through its inner `Builder` class, which is obtained via the static `builder()` factory method. The entity responsible for composing a world generation pipeline (e.g., a BiomeGenerator) is the typical owner.
-   **Scope:** The lifetime of a MaskShader is tied to the shader graph it belongs to. It is typically created during a generator's initialization phase and persists until the generator itself is discarded. It is not a long-lived, session-wide service.
-   **Destruction:** Cleanup is handled by the standard Java garbage collector. There are no native resources or explicit destruction methods.

## Internal State & Concurrency
-   **State:** The core configuration of a MaskShader (its child shader and mask predicate) is **immutable**, established at creation time via the Builder and stored in final fields. This guarantees that the shader's logical behavior cannot change after instantiation. The internal SeedGenerator is mutable, but is not used by any of the `shade` methods, rendering its state irrelevant to the output.
-   **Thread Safety:** The class is **conditionally thread-safe**. The `shade` methods are pure functions with respect to their inputs and the immutable internal configuration. Multiple threads can safely invoke `shade` on the same MaskShader instance without causing data corruption, as there is no shared mutable state that affects the computation.

**WARNING:** The internal `seedGenerator` field is initialized but is not used in any of the `shade` method implementations. This may be a remnant of a previous design. Rely only on the seeds passed directly into the `shade` methods for deterministic output.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder(Class<T> dataType) | static Builder<T> | O(1) | Factory method to create a new Builder instance for configuration. |
| shade(T current, long seed) | T | O(P + S) | Evaluates the mask. If true, delegates to the child shader; otherwise, returns `current`. P is the complexity of the predicate, S is the complexity of the child shader. |
| shade(T current, long seedA, long seedB) | T | O(P + S) | **WARNING:** This overload ignores `seedA` and `seedB`, delegating to the primary `shade` method with a hardcoded seed of `0L`. |
| shade(T current, long seedA, long seedB, long seedC) | T | O(P + S) | **WARNING:** This overload ignores all seed arguments, delegating to the primary `shade` method with a hardcoded seed of `0L`. |

## Integration Patterns

### Standard Usage
The MaskShader must be constructed using its builder. The standard pattern involves defining a Predicate (the mask) and a child Shader, then composing them.

```java
// Example: A shader that only places grass on "Dirt" blocks.

// 1. Define a child shader that turns any block into Grass.
Shader<Block> grassShader = new BlockTypeShader(BlockTypes.GRASS);

// 2. Define a predicate that checks if a block is Dirt.
Predicate<Block> isDirt = (block) -> block.getType() == BlockTypes.DIRT;

// 3. Build the MaskShader.
Shader<Block> placeGrassOnDirtShader = MaskShader.builder(Block.class)
    .withMask(isDirt)
    .withChildShader(grassShader)
    .build();

// 4. Use the composed shader.
Block result = placeGrassOnDirtShader.shade(currentBlock, worldSeed);
// `result` will be a Grass block if `currentBlock` was Dirt.
// `result` will be the original `currentBlock` otherwise.
```

### Anti-Patterns (Do NOT do this)
-   **Incomplete Builder:** Do not call `build()` on the builder without first providing both a mask and a child shader. This will result in an `IllegalStateException` at runtime.
-   **Relying on Multi-Seed Overloads:** Do not use the `shade(current, seedA, seedB)` or `shade(current, seedA, seedB, seedC)` overloads with the expectation that all seeds will be used for generation. These methods discard the provided seeds and pass `0L` to the underlying `shade` call, which can lead to repetitive or non-deterministic results if the child shader relies on a varied seed.

## Data Pipeline
The MaskShader acts as a conditional gate within a larger data processing pipeline. It does not source or sink data itself, but rather routes it based on a logical condition.

> **Flow (Mask Pass):**
> Input Data (T) -> **MaskShader.test(Predicate)** -> `true` -> ChildShader.shade() -> Output Data (T)

> **Flow (Mask Fail):**
> Input Data (T) -> **MaskShader.test(Predicate)** -> `false` -> Output Data (T) (Unmodified)


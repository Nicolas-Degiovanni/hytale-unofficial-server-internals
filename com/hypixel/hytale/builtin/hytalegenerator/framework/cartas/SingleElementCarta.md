---
description: Architectural reference for SingleElementCarta
---

# SingleElementCarta

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.cartas
**Type:** Transient

## Definition
```java
// Signature
public class SingleElementCarta<R> extends BiCarta<R> {
```

## Architecture & Concepts
The SingleElementCarta is a specialized, highly optimized implementation of the BiCarta interface within the world generation framework. Its primary architectural purpose is to represent an infinite, two-dimensional plane where every coordinate (x, z) returns the exact same value.

This class serves as a memory-efficient alternative to storing a large, uniform grid of data. Instead of allocating memory for each coordinate, it stores only a single reference to the element. This pattern is fundamental for establishing base layers or default states in procedural generation, such as a constant sea level, a uniform biome type for a region, or a default terrain height before noise is applied.

It is a foundational, immutable value object used as an input to more complex generator functions and algorithms.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively through the static factory method `SingleElementCarta.of(element)`. It is typically created on-the-fly by higher-level generator components that require a constant-value input.
- **Scope:** This is a transient object. Its lifetime is scoped to the specific generation task that created it. It does not persist and is intended to be passed by reference within a call stack.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as it is no longer referenced by the active generation process.

## Internal State & Concurrency
- **State:** Immutable. The internal `element` field is private and is set only once during object construction via the static factory method. There are no methods to modify this state after creation.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, an instance can be safely shared and read by multiple generator worker threads simultaneously without any need for locks or synchronization. The `apply` method is a pure function, guaranteeing deterministic output.

## API Surface
The public API is minimal, focusing on creation and data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(R element) | SingleElementCarta | O(1) | **Static Factory.** The sole entry point for creating a new instance. |
| apply(int x, int z, WorkerIndexer.Id id) | R | O(1) | Returns the single stored element, completely ignoring the x, z, and id parameters. |
| allPossibleValues() | List<R> | O(1) | Returns an immutable list containing only the single stored element. |

## Integration Patterns

### Standard Usage
SingleElementCarta is used as a constant input for generator functions that operate on the BiCarta abstraction. It provides a simple way to represent a uniform layer.

```java
// Create a carta representing a solid layer of stone
BiCarta<BlockState> stoneLayer = SingleElementCarta.of(STONE_BLOCK);

// Pass this uniform layer to a more complex generator
// that might, for example, carve caves into it.
worldGenerator.applyCaveCarving(stoneLayer, chunkCoords);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the use of the `of()` factory method. Attempting to instantiate it via reflection will break the intended contract.
- **Using Mutable Elements:** A critical design consideration. While the SingleElementCarta itself is immutable, if the generic type `R` is a mutable object, its internal state can be changed. This can lead to severe, difficult-to-debug race conditions in a multithreaded environment.

   **WARNING:** If a mutable object is used as the element, changes made by one thread will be visible to all other threads referencing the same SingleElementCarta instance, violating the expectation of a constant, predictable input. Always prefer using immutable objects for the element type `R`.

## Data Pipeline
SingleElementCarta acts as a data *source* in a generation pipeline. It does not transform data; it provides a constant, uniform block of data for other components to process.

> Flow:
> **SingleElementCarta.of(value)** -> Generator Function (e.g., Noise Applicator, Biome Blender) -> Modified BiCarta -> World Chunk Data


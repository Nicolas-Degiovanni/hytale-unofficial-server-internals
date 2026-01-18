---
description: Architectural reference for TriCarta
---

# TriCarta<R>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class TriCarta<R> {
```

## Architecture & Concepts

The TriCarta class is an abstract, function-like component that serves as a foundational element within the Hytale world generation framework. Its name is a portmanteau of *Tri* (for its three-dimensional integer inputs) and *Carta* (Latin for map), accurately describing its core purpose: to define a deterministic mapping from a 3D world coordinate to a specific, categorical outcome of type R.

Architecturally, TriCarta embodies the **Strategy Pattern** for spatial evaluation. It provides a formal contract for any algorithm that needs to produce a value based on a world position. The world generator does not know or care about the specific implementation details; it only interacts with the TriCarta interface. This allows for immense flexibility, as different TriCarta implementations can be composed, chained, or swapped to create complex and varied world generation pipelines.

The presence of the WorkerIndexer.Id parameter in the `apply` method is a critical design indicator. It signals that TriCarta instances are intended for execution within a highly parallelized, multi-threaded environment. The world generator dispatches chunk generation tasks to a pool of worker threads, and each thread invokes the `apply` method for the coordinates within its assigned region.

Furthermore, the `allPossibleValues` method constrains the output of any implementation to a finite, enumerable set. This is a deliberate design choice that distinguishes TriCarta from continuous-value generators like noise functions. It is used for selecting from a discrete set of possibilities, such as biomes, structure types, or material palettes.

## Lifecycle & Ownership

-   **Creation:** Concrete implementations of TriCarta are not created on-demand. They are instantiated and configured once during the bootstrap phase of the world generator, typically assembled by a higher-level system like a GeneratorGraph or PhaseManager based on world configuration.
-   **Scope:** A TriCarta instance is a long-lived, stateless service object. Its lifetime is bound to the world generator instance itself. It persists for the entire duration of a world generation session.
-   **Destruction:** The object is marked for garbage collection when the parent world generator is shut down and all references to it are released. No explicit destruction logic is required.

## Internal State & Concurrency

-   **State:** Implementations are **strongly expected to be immutable**. The `apply` method should be a pure function, meaning its output depends solely on its inputs. Any internal state, such as a world seed or a lookup table, must be injected at construction time and remain constant throughout the object's lifecycle.

-   **Thread Safety:** The abstract class itself provides no thread safety guarantees. Concrete implementations **must be unconditionally thread-safe**. The `apply` method is guaranteed to be invoked concurrently from multiple worker threads. The safest and most performant implementations are entirely stateless, eliminating the need for locks or other synchronization primitives.

    **WARNING:** Introducing shared mutable state into a TriCarta implementation without proper, highly-performant synchronization will lead to severe, non-deterministic world generation bugs and race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(int, int, int, WorkerIndexer.Id) | R | *Varies* | The core evaluation function. Maps a 3D coordinate to a result of type R. May return null. |
| allPossibleValues() | List<R> | O(1) | Returns a definitive, pre-computed list of all possible non-null values the `apply` method can produce. |

## Integration Patterns

### Standard Usage

Developers do not typically invoke a TriCarta directly. Instead, they create a concrete implementation of the abstract class, defining a specific piece of generation logic. The world generation framework then consumes this implementation, calling its `apply` method millions of times across many threads.

```java
// Example: A simple TriCarta for determining a biome based on altitude.
public final class BiomeCarta extends TriCarta<Biome> {
    private final Biome PLAINS = new Biome("plains");
    private final Biome MOUNTAINS = new Biome("mountains");
    private final List<Biome> ALL_BIOMES = List.of(PLAINS, MOUNTAINS);

    @Override
    public Biome apply(int x, int y, int z, @Nonnull WorkerIndexer.Id workerId) {
        // This is a stateless, deterministic calculation.
        if (y > 128) {
            return MOUNTAINS;
        }
        return PLAINS;
    }

    @Override
    public List<Biome> allPossibleValues() {
        // Returns a pre-computed, immutable list.
        return ALL_BIOMES;
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Mutable State:** The most critical anti-pattern is introducing state that is modified by the `apply` method. This breaks the functional contract and will cause catastrophic race conditions in a parallel environment.

-   **Expensive `allPossibleValues`:** This method may be called during the generator's validation or setup phase. It should not perform any heavy computation. The list of possible values should be a pre-computed constant.

-   **Ignoring Determinism:** A TriCarta must be perfectly deterministic based on its inputs. Relying on non-deterministic sources like `Math.random()` or the system clock will result in inconsistent world generation, visible as chunk borders or seams.

## Data Pipeline

The TriCarta is a key processing stage within the larger world generation data pipeline. It acts as a transformation function, converting raw coordinate data into higher-level game concepts like biomes or material choices.

> Flow:
> Chunk Generation Task -> Worker Thread -> **TriCarta.apply(x, y, z)** -> Categorical Result (e.g., Biome ID) -> Voxel Placement Logic -> Final Chunk Data


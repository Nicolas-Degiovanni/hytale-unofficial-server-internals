---
description: Architectural reference for FieldFunctionOccurrencePositionProvider
---

# FieldFunctionOccurrencePositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient

## Definition
```java
// Signature
public class FieldFunctionOccurrencePositionProvider extends PositionProvider {
```

## Architecture & Concepts
The FieldFunctionOccurrencePositionProvider is a fundamental component in the procedural world generation pipeline, acting as a probabilistic filter for spatial coordinates. It implements the **Decorator** pattern, wrapping another PositionProvider to selectively discard candidate positions based on a continuous probability field.

Its primary role is to translate a conceptual map—represented by a Density function—into concrete occurrences. For example, a Density function describing forest density can be used by this class to decide *exactly where* individual trees should be placed. For each position supplied by the wrapped provider, this class samples the Density function to get a probability value (from 0.0 to 1.0). It then performs a deterministic random check against this probability to decide whether to keep or discard the position.

This mechanism is critical for creating natural-looking, non-uniform distributions of features in the game world. The randomness is seeded by the position's coordinates, ensuring that world generation is fully deterministic and reproducible.

### Lifecycle & Ownership
- **Creation:** Instantiated and configured during the setup of a specific world generation stage. It is constructed with its two key dependencies: the source PositionProvider that generates candidate locations, and the Density function that defines the probability of occurrence at any given location.
- **Scope:** The object's lifetime is typically confined to the execution of a single generation task or stage. It is not a global service and is not intended to be shared across unrelated generation processes.
- **Destruction:** It becomes eligible for garbage collection as soon as the generation stage that created it completes its work and releases its reference.

## Internal State & Concurrency
- **State:** The object is stateful, but its state is immutable after construction. The fields for the Density, the wrapped PositionProvider, and the SeedGenerator are assigned once and are not modified during the object's lifetime. They represent the static configuration for a filtering operation.
- **Thread Safety:** This class is not inherently thread-safe. Its safety is dependent on the thread safety of the injected Density and PositionProvider dependencies. The standard world generation architecture avoids concurrency issues by instantiating a separate generator pipeline, including a new instance of this class, for each worker thread. The `positionsIn` method is re-entrant as all per-call state is managed through the passed-in Context object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| positionsIn(Context context) | void | O(N * D) | Processes candidate positions from the wrapped provider. N is the number of input positions and D is the complexity of the Density.process method. For each position, it invokes the context's consumer if a deterministic random check passes. |

## Integration Patterns

### Standard Usage
This class is designed to be chained with other providers. A developer first defines a source of positions (like a grid) and a probability map (a Density), then uses this class to combine them.

```java
// 1. Define a source of candidate positions (e.g., a simple grid)
PositionProvider sourcePositions = new GridPositionProvider(/*...config...*/);

// 2. Define the probability field (e.g., using Perlin noise)
Density forestDensity = new PerlinNoiseDensity(/*...config...*/);

// 3. Create the filter, wrapping the source with the density function
PositionProvider treePositions = new FieldFunctionOccurrencePositionProvider(
    forestDensity,
    sourcePositions,
    worldSeed
);

// 4. Execute the pipeline to get the final positions
treePositions.positionsIn(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Complex Chaining:** Avoid chaining multiple FieldFunctionOccurrencePositionProvider instances in a deep hierarchy. The computational complexity is multiplicative and can severely degrade world generation performance. It is often more efficient to combine multiple probability fields into a single, more complex Density function.
- **Non-Deterministic Dependencies:** Never provide a Density function or a source PositionProvider that exhibits non-deterministic behavior. The entire world generation system relies on reproducibility, which this class helps enforce through its seeded random number generator. Introducing external randomness would break this contract.

## Data Pipeline
The flow of data through this component is a classic filter pattern. It consumes a stream of positions and produces a thinned-out, probabilistically determined subset of those positions.

> Flow:
> Source PositionProvider -> Candidate Position -> Density.process() -> Probability Value [0.0, 1.0] -> **FieldFunctionOccurrencePositionProvider (Random Check)** -> Accepted Position -> Context Consumer


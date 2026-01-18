---
description: Architectural reference for HeightCondition
---

# HeightCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public class HeightCondition implements ICoordinateRndCondition {
```

## Architecture & Concepts
The HeightCondition is a predicate component within the procedural generation engine. It implements the ICoordinateRndCondition contract, signifying its role as a conditional check evaluated at a specific world coordinate. Its purpose is to determine whether a procedural operation, such as block placement or feature spawning, should proceed based on a height-dependent value.

Architecturally, this class is a classic implementation of the **Strategy Pattern**. It decouples the high-level conditional logic from the specific algorithm used to calculate a height threshold. The core evaluation logic resides within the HeightCondition, but the actual calculation of the threshold value is delegated to an injected IHeightThresholdInterpreter dependency. This design allows for extreme flexibility; the same HeightCondition can be configured to work with noise maps, mathematical curves, or simple constant values by simply providing a different interpreter implementation.

This component is fundamental for creating verticality and layering in world generation, enabling rules like "trees only grow below Y=80" or "ores become more common deeper underground".

### Lifecycle & Ownership
-   **Creation:** Instances are created by higher-level procedural systems, typically when parsing and constructing a graph of generation rules from configuration. A specific IHeightThresholdInterpreter strategy must be provided at construction.
-   **Scope:** The lifetime of a HeightCondition object is scoped to the procedural rule it is part of. It is generally short-lived and does not persist beyond the generation of a specific world chunk or region.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and is eligible for collection as soon as the parent procedural task completes and releases its references.

## Internal State & Concurrency
-   **State:** **Immutable**. The class itself is stateless. Its single field, the interpreter, is marked as final and is injected during construction. The behavior of a HeightCondition instance is fixed for its entire lifetime.
-   **Thread Safety:** **Conditionally Thread-Safe**. The thread safety of this class is entirely inherited from the injected IHeightThresholdInterpreter. If the provided interpreter is thread-safe, then HeightCondition can be safely used across multiple world-generation threads. The eval method itself introduces no mutable state.
    -   **WARNING:** The `java.util.Random` instance passed to the eval method is **not** thread-safe. The calling system is responsible for ensuring that each worker thread uses its own unique or thread-local Random instance to prevent contention and ensure deterministic output.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(seed, x, z, y, random) | boolean | O(I) | Evaluates the condition for a given 3D coordinate. The complexity is determined by the injected interpreter's getThreshold method. Returns true if the interpreter's threshold passes a probabilistic check. |

## Integration Patterns

### Standard Usage
The HeightCondition is intended to be used as a predicate within a larger procedural generation rule set. It is constructed with a pre-configured interpreter and then evaluated for many different coordinates.

```java
// 1. Obtain or create a specific interpreter strategy
IHeightThresholdInterpreter noiseInterpreter = new NoiseBasedInterpreter(...);

// 2. Construct the condition with the desired strategy
ICoordinateRndCondition condition = new HeightCondition(noiseInterpreter);

// 3. In the world generation loop, evaluate the condition
for (int y = 0; y < 256; y++) {
    boolean canPlaceHere = condition.eval(worldSeed, blockX, blockZ, y, threadLocalRandom);
    if (canPlaceHere) {
        // ... perform procedural action
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Interpreters:** Do not inject an IHeightThresholdInterpreter that contains mutable state if the HeightCondition will be used concurrently. This will introduce severe race conditions and result in non-deterministic, unpredictable world generation. Interpreters should be immutable or use appropriate thread-safe mechanisms.
-   **Shared Random Instance:** Never pass the same `java.util.Random` object to calls of eval from different threads. This is a critical threading violation that will cause world generation artifacts.

## Data Pipeline
The HeightCondition acts as a gate or filter within a larger data processing pipeline for world generation. It does not transform data but rather produces a boolean that determines the subsequent path of execution.

> Flow:
> Generation Coordinator -> Provides (Seed, Coordinates, Random) -> **HeightCondition.eval()** -> Boolean Decision -> [Place Feature | Discard Feature]


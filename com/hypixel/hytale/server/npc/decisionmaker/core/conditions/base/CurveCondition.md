---
description: Architectural reference for CurveCondition
---

# CurveCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions.base
**Type:** Component (Abstract Base)

## Definition
```java
// Signature
public abstract class CurveCondition extends Condition {
```

## Architecture & Concepts
The CurveCondition is an abstract base class that forms a fundamental building block of the server-side NPC Utility AI system. Its primary architectural purpose is to decouple the measurement of a specific game state from the evaluation of that state's desirability.

This is achieved by implementing a two-stage evaluation process:
1.  **Input Normalization:** A concrete subclass (e.g., a hypothetical DistanceCondition) is responsible for calculating a raw, normalized value between 0.0 and 1.0 that represents a specific world state. This is handled by the abstract method getNormalisedInput.
2.  **Utility Mapping:** The CurveCondition then takes this normalized input and maps it to a final utility score using a **ResponseCurve** asset. This allows game designers to visually sculpt the "desirability" of a condition in data files without altering game code.

This class is a data-driven component, configured and instantiated entirely through the Hytale Codec system. The static ABSTRACT_CODEC field defines the deserialization logic, including the critical step of resolving the string-based response curve name into a direct asset reference for high-performance lookups during runtime.

## Lifecycle & Ownership
-   **Creation:** Instances of CurveCondition subclasses are never created directly via the new keyword. They are instantiated by the engine's Codec system when loading NPC behavior definitions from asset files. The static ABSTRACT_CODEC acts as the factory during this deserialization process.
-   **Scope:** An instance of a CurveCondition persists for the lifetime of its parent behavior definition. It is a stateless evaluator whose configuration is loaded once and reused for every evaluation cycle.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the associated NPC behavior assets, such as during a world unload or a hot-reload of game data.

## Internal State & Concurrency
-   **State:** The internal state consists of the responseCurve name (a String) and the resolved responseCurveReference. This state is populated once during the afterDecode phase of deserialization and is considered **immutable** thereafter. The class does not modify its own state during utility calculations.
-   **Thread Safety:** This class is **not inherently thread-safe** but is safe within its intended operational context. The NPC decision-making system is expected to operate within a single-threaded game loop per world or region. Invoking calculateUtility on a shared instance from multiple threads simultaneously without external locking will lead to undefined behavior, particularly with the passed CommandBuffer.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateUtility(...) | double | O(C) | Calculates the final utility score. Throws IllegalStateException if the ResponseCurve asset is not found. Complexity depends on the curve's implementation. |
| getNormalisedInput(...) | abstract double | O(N) | Must be implemented by subclasses to provide a normalized (0.0 to 1.0) input value based on the current game state. |
| getResponseCurve() | String | O(1) | Returns the string identifier of the configured ResponseCurve asset. |
| getSimplicity() | int | O(1) | Returns a constant value representing the computational cost, used by the decision-making scheduler. |

## Integration Patterns

### Standard Usage
Direct invocation of a CurveCondition is rare. The class is designed to be implemented by concrete conditions which are then referenced within an NPC's behavior asset file. The system's DecisionMaker will automatically find and evaluate it.

A developer's primary interaction is to create a new subclass.

```java
// 1. Define a concrete condition
public class MyProximityCondition extends CurveCondition {
    // Codec definition for this new type would go here...

    @Override
    protected double getNormalisedInput(int selfIndex, ArchetypeChunk<EntityStore> chunk, ...) {
        // Logic to calculate distance to a target and normalize it to a 0.0-1.0 range.
        // For example, 1.0 if very close, 0.0 if very far.
        double distance = ...;
        double maxDistance = 100.0;
        return 1.0 - MathUtil.clamp(distance / maxDistance, 0.0, 1.0);
    }
}

// 2. Reference it in a data file (e.g., JSON)
// {
//   "type": "MyProximityCondition",
//   "Curve": "LinearInverse"
// }
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new MyProximityCondition()`. This bypasses the Codec system, leaving the internal responseCurveReference field null and causing a NullPointerException during `calculateUtility`.
-   **State Modification:** Do not attempt to modify the responseCurve field after creation. The asset reference is cached during decoding and will not be updated.
-   **Invalid Input Range:** Implementations of getNormalisedInput **must** ensure their output is clamped or naturally falls within the 0.0 to 1.0 range. Values outside this range may produce unexpected results from the ResponseCurve.

## Data Pipeline
The flow of data from configuration to evaluation is a critical aspect of this system's design.

> Flow:
> NPC Behavior Asset File -> Asset Manager -> **ABSTRACT_CODEC** -> CurveCondition Instance -> Decision Maker Tick -> **getNormalisedInput** (reads World State) -> **calculateUtility** (uses ResponseCurve) -> Final Utility Score


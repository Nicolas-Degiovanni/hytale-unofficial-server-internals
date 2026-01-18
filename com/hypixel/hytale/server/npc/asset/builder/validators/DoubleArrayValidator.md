---
description: Architectural reference for DoubleArrayValidator
---

# DoubleArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class DoubleArrayValidator extends Validator {
```

## Architecture & Concepts
The DoubleArrayValidator is an abstract base class that establishes a formal contract for data validation logic specifically targeting arrays of double-precision floating-point numbers. It serves as a foundational component within the server-side NPC asset compilation pipeline.

This class embodies the **Strategy Pattern**, allowing the core asset building system to remain agnostic of specific validation rules. Concrete implementations of this class (e.g., a validator to ensure an array has exactly three elements for a 3D vector) are invoked by the NpcAssetBuilder during the parsing and validation stage. Its primary architectural role is to enforce data integrity and provide a "fail-fast" mechanism, preventing malformed or nonsensical numerical data from being loaded into the game engine, which could otherwise lead to runtime exceptions or unpredictable physics behavior.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated dynamically by the asset building framework, typically through reflection or a pre-configured factory, during the validation phase of a specific NPC asset file.
- **Scope:** The lifecycle of a DoubleArrayValidator instance is extremely brief and transient. It exists only for the duration of a single validation check on a specific field within an asset.
- **Destruction:** The object is eligible for garbage collection immediately after its `test` method is called and a result is returned. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** This abstract class is stateless. Concrete implementations are **strongly expected** to be stateless as well. The validation logic should be a pure function, with the output depending solely on the input array.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. It can be safely used in a multi-threaded asset compilation pipeline where multiple NPC files are processed in parallel.

## API Surface
The public contract is designed for a simple, single-purpose interaction: test an array and provide a descriptive error if it fails.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(double[] var1) | boolean | O(N) | Abstract method. Executes the core validation logic against the input array. Returns true if valid, false otherwise. |
| errorMessage(double[] var1, String var2) | String | O(1) | Abstract method. Generates a detailed, context-aware error message for a failed validation. The String var2 typically contains the name of the asset field being validated. |
| errorMessage(double[] var1) | String | O(1) | Abstract method. Generates a generic error message for a failed validation when extended context is not available. |

## Integration Patterns

### Standard Usage
Developers should extend this class to implement a specific, reusable validation rule. The asset system will then apply this rule during asset loading.

```java
// Example: A concrete validator ensuring an array represents a 2D coordinate.
public class Vec2Validator extends DoubleArrayValidator {
    @Override
    public boolean test(double[] input) {
        return input != null && input.length == 2;
    }

    @Override
    public String errorMessage(double[] input, String fieldName) {
        return "Field '" + fieldName + "' must be a 2-element double array for a Vec2.";
    }

    @Override
    public String errorMessage(double[] input) {
        return "Input must be a 2-element double array.";
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not add member variables to subclasses that store state between calls. Each validation must be an independent, deterministic operation.
- **Direct Invocation:** Avoid calling validator instances directly in game logic. These classes are designed exclusively for the asset compilation pipeline and should be invoked by the asset builder.
- **Complex Logic in Constructors:** Constructors of subclasses should be empty and perform no logic. All work must be done within the `test` method.

## Data Pipeline
This component acts as a gatekeeper in the data flow from raw asset files to in-memory game objects.

> Flow:
> NPC Definition File (JSON/HOCON) -> Asset Parser -> NpcAssetBuilder -> **DoubleArrayValidator Subclass** -> (Success) Validated Game Data OR (Failure) Compilation Error Log


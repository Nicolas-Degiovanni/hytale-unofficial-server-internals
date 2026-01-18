---
description: Architectural reference for ConditionParameter
---

# ConditionParameter

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Data Type / Enumeration

## Definition
```java
// Signature
public enum ConditionParameter {
```

## Architecture & Concepts
ConditionParameter is a type-safe enumeration that defines the fundamental modes of operation for spatial checks within the world generator's material provider system. It serves as a strict contract between world generation configuration files and the procedural generation engine.

Its primary role is not to perform logic, but to represent a fixed set of choices that a designer or modder can specify in data files. The presence of a static CODEC field is the most critical architectural feature, indicating that this enum is a deserializable component in Hytale's data-driven design. It allows human-readable strings in configuration files, such as SPACE_ABOVE_FLOOR, to be safely converted into a compile-time constant, preventing errors from typos or invalid values.

This enum is a leaf node in the configuration tree, providing a specific parameter to a parent component, likely a more complex condition evaluator that measures vertical clearance in the world grid.

### Lifecycle & Ownership
-   **Creation:** Instances are created and managed exclusively by the Java Virtual Machine. The two constants, SPACE_ABOVE_FLOOR and SPACE_BELOW_CEILING, are instantiated once when the ConditionParameter class is loaded by the class loader.
-   **Scope:** Application-wide static constants. These instances persist for the entire lifetime of the game client or server process.
-   **Destruction:** The enum instances are garbage collected when the application terminates and its class loader is unloaded. Manual destruction is neither possible nor necessary.

## Internal State & Concurrency
-   **State:** Inherently immutable. Enum constants are final static instances, and their internal state cannot be modified after creation. They represent fixed concepts.
-   **Thread Safety:** Fully thread-safe. As immutable singletons, instances of ConditionParameter can be safely accessed, passed, and read by any number of threads without locks or synchronization. This is essential for the highly parallelized world generation system.

## API Surface
The public contract consists of the predefined enum constants and the static codec for data binding.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SPACE_ABOVE_FLOOR | ConditionParameter | O(1) | A constant representing a condition that checks for open space vertically upwards from a given block. |
| SPACE_BELOW_CEILING | ConditionParameter | O(1) | A constant representing a condition that checks for open space vertically downwards from a given block. |
| CODEC | Codec<ConditionParameter> | O(1) | A static service used by the engine to serialize and deserialize the enum from configuration files. |

## Integration Patterns

### Standard Usage
This enum is not intended to be used directly in procedural code. Instead, it is deserialized from configuration data as part of a larger condition object. Engine systems then use the deserialized value to drive conditional logic.

```java
// A hypothetical system that consumes a condition containing this enum
public class MaterialPlacer {

    public void applyMaterial(WorldGenContext context, SpaceCondition condition) {
        // The 'condition' object would have been loaded from a JSON file,
        // with its 'parameter' field deserialized via the CODEC.
        switch (condition.getParameter()) {
            case SPACE_ABOVE_FLOOR:
                // Execute logic for checking space above a surface
                break;
            case SPACE_BELOW_CEILING:
                // Execute logic for checking space below a surface
                break;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **String Comparison:** Never compare an enum instance to a string using `toString()`. This is brittle, slow, and defeats the purpose of type safety. Always use direct equality `==` or a switch statement.
-   **Null Checks:** A deserialized enum from a valid configuration should never be null. If it is, this indicates a critical failure in the data loading pipeline or a malformed configuration file. Code should treat a null value as an exceptional error state.

## Data Pipeline
ConditionParameter acts as a validated, type-safe data point that flows from configuration into the core world generation logic.

> Flow:
> WorldGen JSON File -> Hytale Codec Engine -> **ConditionParameter** (Instance) -> SpaceAndDepthCondition -> MaterialProvider Logic


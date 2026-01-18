---
description: Architectural reference for NoisePropertyType
---

# NoisePropertyType

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Enumeration

## Definition
```java
// Signature
public enum NoisePropertyType {
```

## Architecture & Concepts
The NoisePropertyType enumeration defines a fixed set of operations or transformations that can be applied within the procedural noise generation pipeline. It serves as a type-safe vocabulary for constructing complex noise graphs, acting as the set of "opcodes" that the noise engine can execute.

Each constant in this enum represents a distinct mathematical or structural modification to a noise field, such as scaling, blending, or distortion. By using this enum, the system ensures that noise graph definitions, whether loaded from configuration files or constructed in code, are valid and constrained to a known set of behaviors. This prevents ambiguity and runtime errors associated with string-based or integer-based type identifiers.

Architecturally, NoisePropertyType is a foundational, static component of the procedural library. It decouples the definition of a noise graph from its implementation, allowing the engine to interpret and execute a series of these operations to produce the final procedural output.

## Lifecycle & Ownership
- **Creation:** Enum instances are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. They are compile-time constants.
- **Scope:** The scope is static. All instances exist for the entire lifetime of the application.
- **Destruction:** Instances are unloaded when the JVM shuts down. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** NoisePropertyType instances are immutable. They contain no state other than their own definition (name and ordinal).
- **Thread Safety:** This enumeration is inherently thread-safe. As immutable, static constants, its values can be safely accessed and compared from any thread without synchronization. This is a critical feature for a parallelized world generation system.

## API Surface
The public API consists of the enumeration constants themselves. Each constant represents a specific node type or operation within a noise generation graph.

| Constant | Description |
| :--- | :--- |
| DISTORTED | Applies a secondary noise source to displace the coordinates of the primary source, creating warping effects. |
| MAX | Takes the maximum value from two or more input noise sources at each point. |
| MIN | Takes the minimum value from two or more input noise sources at each point. |
| MULTIPLY | Multiplies the values of two or more input noise sources together. |
| NORMALIZE | Remaps the noise output to a specific range, typically 0.0 to 1.0. |
| FORMULA | Applies a custom mathematical formula or expression to the noise values. |
| SCALE | Multiplies the noise output by a scalar value, affecting its amplitude. |
| SUM | Adds the values of two or more input noise sources together. |
| INVERT | Inverts the noise output, typically calculated as (1.0 - value). |
| OFFSET | Adds a constant value to the noise output, shifting the entire range. |
| ROTATE | Applies a rotational transformation to the input coordinates before sampling the noise. |
| GRADIENT | Applies a gradient ramp, remapping noise values based on a defined color or value gradient. |
| CURVE | Remaps noise values along a user-defined curve, allowing for non-linear adjustments to the distribution. |
| BLEND | Combines two or more noise sources using a third noise source as a mask or alpha channel. |

## Integration Patterns

### Standard Usage
NoisePropertyType is primarily used in control flow statements, such as a switch, to dispatch to the correct noise processing implementation. It is retrieved from a data structure that defines a node in the noise graph.

```java
// How the noise engine processes a graph node
NoiseGraphNode node = ...;
NoisePropertyType operation = node.getType();

switch (operation) {
    case SCALE:
        // Apply scaling logic
        processScaleNode(node);
        break;
    case BLEND:
        // Apply blending logic
        processBlendNode(node);
        break;
    // ... other cases
    default:
        throw new UnsupportedOperationException("Unknown noise property type: " + operation);
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the integer ordinal of the enum for logic or serialization. The order of constants may change, breaking saved data and conditional logic. Always compare the enum instances directly.
- **String Comparison:** Do not serialize or reference the enum by a loose string. Use the built-in `name()` or `valueOf()` methods to ensure type safety and prevent typos.

## Data Pipeline
This enum does not process data itself; it directs the flow of data within the noise engine. It is a control signal, not a data container.

> Flow:
> Noise Graph Asset (JSON/XML) -> Asset Deserializer -> NoiseGraphNode(type: **NoisePropertyType**) -> Noise Generation Engine -> Final Noise Field


---
description: Architectural reference for NoiseFormulaProperty
---

# NoiseFormulaProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class NoiseFormulaProperty implements NoiseProperty {
```

## Architecture & Concepts
The NoiseFormulaProperty is a fundamental component within the procedural generation library, acting as a modifier in the noise generation pipeline. It is a direct implementation of the **Decorator** design pattern, designed to wrap an existing NoiseProperty and apply a post-processing mathematical transformation to its output.

This architectural choice promotes composition over inheritance, allowing for the creation of complex noise behaviors by chaining together simpler, reusable components. For instance, a standard Perlin noise generator can be transformed into ridged multi-fractal noise by wrapping it with a NoiseFormulaProperty configured with the RIDGED formula, without altering the original Perlin implementation.

The available transformations are provided by the nested NoiseFormula enum, which serves as a registry of predefined **Strategies**. Each enum constant encapsulates a specific mathematical function (e.g., INVERTED, SQUARED, SQRT), decoupling the transformation logic from the decorator itself.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by higher-level systems responsible for constructing a procedural generation graph, such as a world or biome generator. It is not a managed service and is instantiated directly via its constructor.
- **Scope:** The lifetime of a NoiseFormulaProperty instance is strictly tied to its parent object, typically a larger configuration or generator instance. It is a value-like object, intended to be short-lived relative to the application session.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the noise graph or generator that references it is dereferenced. No explicit cleanup or destruction methods are required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields for the wrapped NoiseProperty and the selected Formula are declared as final and are initialized only once during construction. The class holds no mutable state.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable and stateless design, a single NoiseFormulaProperty instance can be safely shared and invoked by multiple threads simultaneously without any need for external synchronization or locks. This is a critical feature for enabling parallelized world generation, where multiple workers may need to sample the same noise function concurrently.

## API Surface
The public API is minimal, adhering to the contract defined by the NoiseProperty interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(1) | Retrieves the value from the wrapped property and applies the configured formula. |
| get(seed, x, y, z) | double | O(1) | Retrieves the value from the wrapped property and applies the configured formula. |

## Integration Patterns

### Standard Usage
The primary use case is to wrap an existing NoiseProperty to modify its output. This is a core pattern for building up complex procedural assets.

```java
// 1. Obtain a base noise source
NoiseProperty baseNoise = new SimplexNoiseProperty(/*...args*/);

// 2. Wrap the base noise with a formula to create a new behavior
//    Here, we create ridged noise by applying an absolute value function.
NoiseProperty ridgedNoise = new NoiseFormulaProperty(
    baseNoise,
    NoiseFormula.RIDGED_FIX.getFormula()
);

// 3. Use the decorated property in the generation algorithm
//    The value is now transformed from the standard [-1, 1] range
//    to a [0, 1] range with a sharp ridge at the zero-crossing.
double value = ridgedNoise.get(seed, x, y);
```

### Anti-Patterns (Do NOT do this)
- **Null Wrapping:** Never pass a null NoiseProperty to the constructor. The class performs no null checks, and any subsequent call to a get method will result in a fatal NullPointerException.
- **Excessive Chaining:** While chaining multiple NoiseFormulaProperty instances is technically possible (e.g., wrapping a wrapper), it can lead to code that is difficult to debug and understand. For highly complex transformations, consider implementing a custom Formula instead of deep nesting.

## Data Pipeline
NoiseFormulaProperty acts as a stateless processing node within a larger data flow. It does not source or sink data but transforms it as it passes through.

> Flow:
> Upstream NoiseProperty.get() -> **NoiseFormulaProperty** (applies formula) -> Downstream Consumer (e.g., TerrainGenerator, Voxel Carver)


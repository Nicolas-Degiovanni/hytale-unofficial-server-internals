---
description: Architectural reference for ConstantNoise
---

# ConstantNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class ConstantNoise implements NoiseFunction {
```

## Architecture & Concepts
The ConstantNoise class is a foundational component within the procedural generation library. It represents the most basic implementation of the NoiseFunction interface, providing a uniform, unchanging value across all spatial coordinates.

Architecturally, it serves as a "leaf node" or a "null object" in a composite tree of noise operations. More complex noise functions, such as blending or selection nodes, can use a ConstantNoise instance as a predictable input. For example, it can be used to define a flat water table, a solid bedrock layer, or a default value in a conditional generation step. Its primary role is to provide a non-variant baseline from which more complex procedural features can be built.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor, `new ConstantNoise(value)`. This is typically done by higher-level systems that parse world generation configurations or build procedural graphs. It is not a managed service.
- **Scope:** The lifetime of a ConstantNoise instance is tied to its owner. It is a lightweight value object, and its scope is typically confined to the procedural generation task that required it. For instance, if a biome definition uses a ConstantNoise for its base height, the instance persists as long as the biome definition is loaded.
- **Destruction:** The object is eligible for garbage collection as soon as all references to it are dropped. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. The internal state consists of a single `final double value`, which is set at construction time and cannot be modified thereafter. All method calls are deterministic and produce no side effects.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, instances can be safely shared and accessed by multiple threads simultaneously without any need for external synchronization or locks. The `get` methods are pure functions.

## API Surface
The public contract is minimal, focusing on the implementation of the NoiseFunction interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ConstantNoise(double) | constructor | O(1) | Creates a new noise function that will always return the specified value. |
| get(int, int, double, double) | double | O(1) | Returns the configured constant value, ignoring all input parameters. |
| get(int, int, double, double, double) | double | O(1) | Returns the configured constant value, ignoring all input parameters. |
| getValue() | double | O(1) | A simple accessor for the configured constant value. |

## Integration Patterns

### Standard Usage
ConstantNoise is used as a concrete implementation wherever a NoiseFunction is required, especially for defining simple, flat surfaces or providing a default input to more complex noise algorithms.

```java
// Example: Defining a flat water level for a terrain generator
import com.hypixel.hytale.procedurallib.NoiseFunction;

NoiseFunction waterLevel = new ConstantNoise(64.0);

// This function can now be passed to other systems
double heightAtPoint = waterLevel.get(seed, 0, 123.4, 567.8); // Always returns 64.0
```

### Anti-Patterns (Do NOT do this)
- **Unnecessary Instantiation:** Do not create a new ConstantNoise instance inside a tight loop. If the same constant value is needed repeatedly, create a single instance and reuse it. Its thread-safe nature makes this pattern safe and efficient.
- **Misuse as a Dynamic Value:** This object is immutable. Do not attempt to use it as a placeholder for a value that is expected to change. It is designed exclusively for static, unchanging values.

## Data Pipeline
ConstantNoise acts as a simple data source node within a larger procedural generation pipeline. It does not process incoming data but rather injects a pre-configured value into the stream.

> Flow:
> Generator Configuration -> **new ConstantNoise(value)** -> Noise Graph Executor -> Voxel Engine -> Final World Data


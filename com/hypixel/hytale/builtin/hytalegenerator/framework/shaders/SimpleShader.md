---
description: Architectural reference for SimpleShader
---

# SimpleShader

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.shaders
**Type:** Transient Value Object

## Definition
```java
// Signature
public class SimpleShader<T> implements Shader<T> {
```

## Architecture & Concepts
The SimpleShader is the most fundamental implementation of the Shader interface within the world generation framework. Its primary architectural role is to provide a constant, non-variant value, acting as a terminal node or a foundational input in a larger, composite shader graph.

Unlike procedural shaders that compute values based on inputs like world coordinates or random seeds, the SimpleShader completely ignores these parameters. It consistently returns a pre-configured, constant value. This makes it an essential building block for defining base materials, default states, or fallback values within the generation pipeline. For example, a more complex `BlendShader` might use two SimpleShader instances to represent "Stone" and "Dirt", blending between them based on a noise function.

Architecturally, it embodies the concept of a constant function in a functional programming paradigm, providing a predictable and idempotent source of data.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `SimpleShader.of(value)`. The constructor is private to enforce this pattern and prevent subclassing or improper instantiation. It is typically created by higher-level generator services or configuration loaders that require a constant shader.
- **Scope:** SimpleShader is a transient object. Its lifetime is bound to the component that creates it, such as a specific biome generator or a material layer definition. It does not persist globally and is intended to be lightweight and frequently created.
- **Destruction:** Managed entirely by the Java Garbage Collector. Once all references to an instance are dropped, it becomes eligible for collection. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** Immutable. The internal `value` field is declared as `final` and is initialized only once during construction. The object's state cannot be modified after creation.
- **Thread Safety:** This class is unconditionally thread-safe. Its immutability guarantees that concurrent read operations from any number of threads will not result in data corruption or race conditions. No synchronization primitives (e.g., locks) are necessary, making it highly performant in multi-threaded world generation contexts.

## API Surface
The public contract is minimal, focusing on creation and evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(value) | static SimpleShader | O(1) | Factory method to construct a new SimpleShader instance. |
| shade(current, seed) | T | O(1) | Returns the configured constant value, ignoring all input parameters. |
| shade(current, seedA, seedB) | T | O(1) | Returns the configured constant value, ignoring all input parameters. |
| shade(current, seedA, seedB, seedC) | T | O(1) | Returns the configured constant value, ignoring all input parameters. |

## Integration Patterns

### Standard Usage
SimpleShader is used as a concrete implementation of the Shader interface, often to define a base case or a constant input for more complex shader compositions.

```java
// Define a shader that always returns a specific block state for "Stone"
Shader<BlockState> stoneShader = SimpleShader.of(STONE_BLOCK_STATE);

// Use it in a higher-level system that expects any Shader
terrainGenerator.setBaseMaterial(stoneShader);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to instantiate it via reflection would violate the class's design contract. Always use the `SimpleShader.of()` factory method.
- **Misuse for Dynamic Values:** Do not wrap a dynamic or mutable object in a SimpleShader. The purpose of this class is to represent an immutable constant. Using it for values that are expected to change based on generator inputs is a logical error and defeats the purpose of the shader system.

## Data Pipeline
SimpleShader acts as a data source, not a processor. It originates a value that is then fed into the rest of the shader pipeline. It does not transform incoming data.

> Flow:
> **SimpleShader (Provides Constant Value)** -> Shader Composition Engine -> World Generator -> Voxel Data Output


---
description: Architectural reference for TriDoubleFunction
---

# TriDoubleFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface TriDoubleFunction<R> {
```

## Architecture & Concepts
TriDoubleFunction is a functional interface that serves as a standardized contract for any operation that accepts three primitive double arguments and produces a result of a generic type R. It is a specialized utility designed to fill a gap in the standard Java functional library (java.util.function), which does not provide a three-argument function primitive for doubles.

Within the Hytale Generator framework, this interface is primarily used to pass mathematical or procedural generation logic as a parameter. It allows algorithms, such as noise generators or biome interpolators, to be decoupled from the specific mathematical formulas they apply. By accepting a TriDoubleFunction, a system can execute arbitrary, user-defined logic on 3D coordinates (x, y, z) or other three-component data without needing to be recompiled.

This pattern promotes a highly modular and extensible design, enabling the composition of complex behaviors from simple, reusable mathematical functions.

## Lifecycle & Ownership
- **Creation:** Instances are not created in the traditional sense. As a functional interface, it is intended to be implemented by a lambda expression or a method reference at the call site. The Java runtime effectively creates an anonymous inner class instance to represent the lambda.
- **Scope:** The lifecycle of a TriDoubleFunction instance is tied to the scope of its implementation. If defined as a lambda within a method, it exists only for the duration of that method's execution and will be garbage collected afterward, unless captured by a longer-lived object.
- **Destruction:** Managed entirely by the Java Garbage Collector. When no more references to the lambda instance exist, it becomes eligible for collection.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, implementations (lambda expressions) can be stateful if they close over (capture) non-final variables from their enclosing scope.

- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of any specific *implementation* is the responsibility of the developer. Lambdas that are purely mathematical and do not modify shared state are thread-safe. Implementations that capture and modify shared, mutable state are **not** thread-safe and must be synchronized externally.

**WARNING:** Avoid creating stateful lambdas for this interface when used in parallelized generator tasks. Unpredictable behavior and race conditions will occur if the captured state is not managed correctly.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(double, double, double) | R | O(N) | Executes the function's logic. Complexity is dependent on the implementation. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda expression to a method that requires a three-dimensional mathematical operation. This is common in procedural generation contexts.

```java
// Example: Using a TriDoubleFunction to define a 3D density function
// for a procedural terrain generator.
public class TerrainGenerator {
    public Voxel a_method_that_uses_the_function(int x, int y, int z, TriDoubleFunction<Float> densityFunc) {
        // Normalize coordinates
        double normX = x / 100.0;
        double normY = y / 100.0;
        double normZ = z / 100.0;

        // The generator invokes the provided logic without knowing its details
        float density = densityFunc.apply(normX, normY, normZ);

        // ... logic to create a voxel based on density
    }
}

// At the call site:
TerrainGenerator generator = new TerrainGenerator();
TriDoubleFunction<Float> simplexNoise = (x, y, z) -> Simplex.noise(x, y, z);
generator.a_method_that_uses_the_function(10, 20, 30, simplexNoise);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations in Parallel Systems:** Do not implement this interface with a lambda that modifies a shared external collection or object when the calling system is multi-threaded. This will lead to severe concurrency issues.
- **Complex Class Implementations:** Avoid creating a full, named class that implements TriDoubleFunction unless the logic is exceptionally complex and requires its own internal state and helper methods. For most use cases, a lambda or method reference is sufficient and more readable.
- **Misuse for Non-Numeric Data:** While technically possible, do not use this interface to pass data that is not semantically a set of three double-precision numbers. This violates the principle of least astonishment and makes the code harder to understand.

## Data Pipeline
This interface does not represent a data pipeline itself, but rather a single, stateless transformation step *within* a larger pipeline. It acts as a "black box" that transforms three input doubles into a single output value.

> Flow:
> 3D Coordinate (double, double, double) -> **TriDoubleFunction.apply()** -> Transformed Value (Type R)


---
description: Architectural reference for Transform
---

# Transform

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations.transform
**Type:** Functional Interface / Strategy Pattern

## Definition
```java
// Signature
public interface Transform {
```

## Architecture & Concepts
The **Transform** interface is a foundational component within the Hytale builder tools framework. It represents a single, atomic, in-place geometric operation on a 3D integer coordinate, **Vector3i**. This interface abstracts the concept of a transformation, allowing different operations like translation, rotation, or scaling to be treated uniformly.

Architecturally, **Transform** embodies the Strategy and Command patterns. It decouples the definition of a transformation from the system that executes it. A higher-level system, such as a selection manipulation tool, does not need to know the specifics of a transformation; it only needs a **Transform** object to which it can pass coordinates.

The interface's design strongly promotes functional composition. The default method **then** allows for the chaining of multiple **Transform** objects into a single, composite operation. This creates a powerful and declarative pipeline for complex geometric manipulations, where a sequence of operations is built first and then applied atomically to a set of points.

## Lifecycle & Ownership
- **Creation:** Instances are typically lightweight and created on-demand. Common creation patterns include implementing the interface with a lambda expression for a specific operation or composing existing instances via the **then** method, which internally uses the **Composite** factory. The static field **NONE** provides a pre-instantiated, singleton no-op implementation.
- **Scope:** The lifetime of a **Transform** object is almost always transient and method-scoped. It is created to execute a specific user-initiated action (e.g., "move selection up by 10 blocks") and is eligible for garbage collection immediately after the operation completes.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this interface or its typical implementations.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Implementations, however, may be stateful. For example, a translation transform might store the translation offset vector. The composite transform created by **then** holds a list of the transforms it must apply in sequence. The provided **NONE** implementation is a stateless immutable singleton.
- **Thread Safety:** The **Transform** contract does not guarantee thread safety. Implementations are **not** assumed to be thread-safe. Operations within the builder tools are expected to occur on a single main thread.

**WARNING:** Do not share stateful **Transform** implementations across threads without explicit synchronization. Doing so will lead to race conditions and unpredictable geometric corruption. Stateless implementations (e.g., a transform that always adds a constant vector) are inherently safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(Vector3i var1) | void | O(N) | Applies the transformation to the vector **in-place**. Complexity is O(1) for a simple transform and O(N) for a composite, where N is the number of chained transforms. |
| then(Transform next) | Transform | O(1) | Composes this transform with another. Returns a new **Composite** transform that will apply this transform first, followed by the next. |
| NONE | Transform | N/A | A static, singleton instance that performs no operation. Useful as a default or base case. |

## Integration Patterns

### Standard Usage
The primary pattern is to define one or more transformations and chain them together to form a final operation. This composite transform is then passed to a system that iterates over a collection of coordinates, applying the transform to each one.

```java
// Example: Create a transform that moves a point and then applies a second operation.
// Assume 'someOtherTransform' is a pre-existing Transform instance.
Transform translation = (vec) -> vec.add(10, 5, 0);
Transform finalOperation = translation.then(someOtherTransform);

// A tool would then use this operation on a selected point
Vector3i blockPosition = new Vector3i(1, 1, 1);
finalOperation.apply(blockPosition); // blockPosition is now modified
```

### Anti-Patterns (Do NOT do this)
- **Ignoring In-Place Modification:** The **apply** method modifies the **Vector3i** object passed to it. Do not assume the original vector is preserved. If you need to preserve the original coordinate, create a copy before calling **apply**.
- **Stateful Lambdas in Concurrent Environments:** Avoid creating lambda implementations that capture and modify external state if the resulting **Transform** could ever be used across multiple threads. This is a direct path to non-deterministic behavior.

## Data Pipeline
**Transform** acts as a processing node in the builder tools' operational flow. It does not handle data streams but rather executes a discrete logic step.

> Flow:
> User Input -> Builder Tool Controller -> **Transform Instantiation & Composition** -> Voxel Selection Iterator -> **transform.apply(position)** -> World State Update Request


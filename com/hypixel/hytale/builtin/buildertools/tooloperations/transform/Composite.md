---
description: Architectural reference for Composite
---

# Composite

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations.transform
**Type:** Transient

## Definition
```java
// Signature
public class Composite implements Transform {
```

## Architecture & Concepts
The Composite class is a concrete implementation of the Composite design pattern, applied to the `Transform` interface. Its primary function is to chain two distinct transformation operations into a single, sequential operation. This allows the builder tools system to combine simple transforms, such as a rotation and a translation, and treat them as a single, atomic `Transform` object.

Architecturally, it serves as a fundamental building block for creating complex geometric manipulations without complicating the client code. Instead of requiring a caller to manage a list of transforms and apply them in order, the system can build a nested tree of Composite transforms that represents the entire operation.

A key architectural feature is the static factory method, `of`. This method acts as an optimization gateway, preventing the creation of redundant or computationally unnecessary Composite objects. It intelligently unwraps or bypasses the composition if one or both of the inputs are a no-op `Transform.NONE`, thus reducing object allocation and simplifying the resulting transform chain.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively via the static factory method `Composite.of(Transform, Transform)`. It is typically created on-the-fly within the builder tools framework whenever a user combines multiple operations.
- **Scope:** Short-lived and stateless. A Composite object's lifetime is typically confined to the execution of a single build tool action. It does not persist across operations.
- **Destruction:** As a transient object with no external state, it becomes eligible for garbage collection as soon as the operation that created it completes and all references to it are dropped.

## Internal State & Concurrency
- **State:** Immutable. The internal `first` and `second` transform references are declared `final` and are set only once during construction. The object's state cannot be modified after it has been created.
- **Thread Safety:** Inherently thread-safe due to its immutability. The `apply` method can be safely invoked from multiple threads concurrently without external locking, assuming the composed `Transform` objects are also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(first, second) | Transform | O(1) | **[Factory]** Combines two transforms. Returns an optimized transform, potentially avoiding a new allocation if one input is a no-op. |
| apply(vector3i) | void | O(T1+T2) | **[Interface]** Sequentially applies the first, then the second transform to the provided vector. The operation is performed in-place. |

## Integration Patterns

### Standard Usage
The `of` factory method should always be used to chain transforms. The framework handles the composition transparently when users apply multiple modifications.

```java
// Assume 'rotation' and 'translation' are existing Transform objects
Transform combined = Composite.of(rotation, translation);

// The combined transform can now be applied as a single operation
Vector3i myVector = new Vector3i(1, 0, 0);
combined.apply(myVector);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private for a reason. Attempting to instantiate `Composite` with `new` would bypass the critical optimization logic in the `of` factory method. Always use `Composite.of()`.
- **Manual Null-Checking:** Do not write code that checks if a transform is `Transform.NONE` before passing it to `Composite.of`. The factory method is designed to handle this case efficiently.

```java
// BAD: Redundant and verbose check
Transform result;
if (secondTransform == Transform.NONE) {
    result = firstTransform;
} else {
    result = Composite.of(firstTransform, secondTransform);
}

// GOOD: Clean and lets the factory handle optimization
Transform result = Composite.of(firstTransform, secondTransform);
```

## Data Pipeline
The Composite class acts as a processor node within the builder tool's data transformation pipeline. It does not generate or terminate data but rather combines other processing nodes.

> Flow:
> User Input -> Builder Operation -> Transform A + Transform B -> `Composite.of(A, B)` -> **Composite** -> `apply(Vector3i)` -> Final Block Position


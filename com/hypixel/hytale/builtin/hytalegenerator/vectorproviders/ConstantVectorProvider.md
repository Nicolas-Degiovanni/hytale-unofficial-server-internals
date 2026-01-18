---
description: Architectural reference for ConstantVectorProvider
---

# ConstantVectorProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.vectorproviders
**Type:** Transient

## Definition
```java
// Signature
public class ConstantVectorProvider extends VectorProvider {
```

## Architecture & Concepts
The ConstantVectorProvider is a foundational implementation of the VectorProvider strategy pattern. It serves as a terminal node or a static input source within the Hytale Generator framework. Its sole responsibility is to supply a fixed, predetermined Vector3d value upon request.

This class is designed for simplicity and predictability. Unlike more complex providers that might calculate vectors based on noise functions, world position, or other contextual data, the ConstantVectorProvider always returns the same value it was configured with at creation. It is frequently used to define static offsets, constant forces like gravity, or fixed directional vectors within a larger procedural generation graph.

Crucially, this provider intentionally ignores the Context object passed to its process method, reinforcing its role as a static, context-insensitive data source.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor, typically during the configuration of a parent system such as a world generator, particle emitter, or animation controller. It is not a managed service and should not be retrieved from a central registry.
-   **Scope:** The lifetime of a ConstantVectorProvider instance is strictly bound to its owner. It persists as a component of the system that created it and holds no global state.
-   **Destruction:** The object is eligible for garbage collection as soon as its owner is dereferenced. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** Effectively immutable. The internal Vector3d state is established at construction time via a defensive copy. Subsequent calls to the process method return a new clone of this internal state, preventing any external mutation.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable state and lack of side effects guarantee that a single instance can be safely shared and accessed by multiple threads concurrently without requiring any external synchronization or locks.

## API Surface
The public API is minimal, consisting only of the constructor and the inherited process method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ConstantVectorProvider(value) | constructor | O(1) | Creates a new provider with a fixed vector. The input is defensively cloned. |
| process(context) | Vector3d | O(1) | Returns a clone of the configured vector. The context parameter is ignored. |

## Integration Patterns

### Standard Usage
The provider is created once with a desired vector and passed to a system that consumes VectorProvider implementations. The consuming system is then responsible for invoking the process method as needed.

```java
// Define a constant upward vector for a particle effect
Vector3d gravity = new Vector3d(0, -9.8, 0);
VectorProvider constantGravityProvider = new ConstantVectorProvider(gravity);

// The particle system will now invoke constantGravityProvider.process() on each tick
particleSystem.setGravitySource(constantGravityProvider);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Do not create a new ConstantVectorProvider inside a game loop or other frequently executed code path. As an immutable object, it should be instantiated once and reused.
-   **Expecting Contextual Behavior:** Do not use this provider with the expectation that the input Context will influence the result. It is designed to be static. If you need context-sensitive vectors, use a different VectorProvider implementation.

## Data Pipeline
As a "provider", this class acts as a source node in a data flow. It does not transform incoming data; it originates it.

> Flow:
> **ConstantVectorProvider** (Ignores Context) -> Generator System -> Final Vector3d Value


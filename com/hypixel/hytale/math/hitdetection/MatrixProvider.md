---
description: Architectural reference for MatrixProvider
---

# MatrixProvider

**Package:** com.hypixel.hytale.math.hitdetection
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface MatrixProvider {
```

## Architecture & Concepts
The MatrixProvider interface defines a fundamental contract for any object that can supply a transformation matrix. Within the engine, this pattern is a cornerstone of the Dependency Inversion Principle, decoupling systems that *perform* spatial calculations (like hit detection or rendering) from the objects that *possess* spatial state (like entities, cameras, or bones in a skeleton).

Instead of a physics or rendering system directly querying an entity's position, rotation, and scale to build a matrix, it simply requests the final, composed matrix via this interface. This abstracts away the complexity of how the matrix is derived, whether it is cached, dirty-checked, or calculated on-the-fly from component parts. Its placement in the `hitdetection` package strongly indicates its primary role is to provide the world-space transformation matrix for collision volumes.

## Lifecycle & Ownership
As an interface, MatrixProvider has no lifecycle of its own. Its lifecycle is entirely dictated by the concrete class that implements it.

- **Creation:** An object implementing MatrixProvider is created according to its own specific logic. For example, an Entity implementing this interface is created by the EntityManager, while a Camera might be created by the rendering system.
- **Scope:** The scope of the implementing object determines the availability of the provided matrix. An entity's provider lives and dies with the entity. A camera's provider persists as long as that camera is active in the scene.
- **Destruction:** The provider becomes invalid when its implementing object is destroyed and garbage collected. Consumers must not hold long-term references to a MatrixProvider without being aware of the owner's lifecycle.

## Internal State & Concurrency
The contract of MatrixProvider implies specific expectations for its implementations regarding state and thread safety.

- **State:** Implementations are expected to manage their own internal state (e.g., position, rotation vectors) to compute the final Matrix4d. The returned matrix should be considered a *snapshot* of the object's state at the moment of the call. It is a critical design requirement that implementations return either an immutable matrix or a new copy to prevent external mutation. Modifying a returned matrix from a provider is a severe anti-pattern that can lead to visual corruption and simulation errors.
- **Thread Safety:** The getMatrix method must be thread-safe. It is frequently called by asynchronous systems, such as the physics thread performing collision checks or a render thread preparing draw calls. Implementations must ensure that internal state is accessed in a thread-safe manner, typically through immutable data structures, atomic operations, or appropriate locking if the underlying state is mutable.

## API Surface
The API is minimal, consisting of a single method that fulfills the entire contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMatrix() | Matrix4d | O(1) to O(N) | Returns the current transformation matrix. Complexity depends on implementation; it may be an O(1) cached lookup or an O(N) recalculation based on a scene graph hierarchy. |

## Integration Patterns

### Standard Usage
The primary pattern is to retrieve a MatrixProvider from an object to perform spatial calculations without being coupled to the object's concrete type. This is common in systems that iterate over collections of heterogeneous objects.

```java
// A hit detection system processing a potential target
void performRaycast(GameObject target, Ray ray) {
    MatrixProvider provider = target.getComponent(MatrixProvider.class);
    if (provider != null) {
        Matrix4d transform = provider.getMatrix();
        // Use the matrix to transform the ray or the object's bounding box
        // into a common coordinate space for intersection testing.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** The most severe anti-pattern is modifying the matrix returned by getMatrix. This breaks the encapsulation of the provider and can cause unpredictable side effects in other systems that also use that provider.

    ```java
    // DO NOT DO THIS
    Matrix4d transform = provider.getMatrix();
    transform.translate(1, 0, 0); // This might modify the provider's internal state!
    ```

- **Stale Caching:** Do not fetch the matrix once and cache it externally for an extended period. The provider's internal state can change on any tick. Always call getMatrix to ensure the most up-to-date transformation is used.

## Data Pipeline
The MatrixProvider serves as a data source, feeding transformation data into more complex systems.

> Flow:
> Entity Component System (updates position/rotation) -> **Concrete MatrixProvider** (caches or computes matrix) -> Hit Detection System (calls getMatrix) -> Collision Resolution


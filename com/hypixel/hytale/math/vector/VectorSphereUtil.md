---
description: Architectural reference for VectorSphereUtil
---

# VectorSphereUtil

**Package:** com.hypixel.hytale.math.vector
**Type:** Utility

## Definition
```java
// Signature
public class VectorSphereUtil {
```

## Architecture & Concepts
VectorSphereUtil is a stateless, high-performance utility class designed for efficient spatial querying. Its primary function is to filter collections of objects based on their proximity to a central point, effectively performing a "point-in-ellipsoid" test for each element. This component is fundamental for game logic that requires proximity checks, such as area-of-effect calculations, entity culling, or AI target acquisition.

The architecture is defined by two key principles:
1.  **Stateless, Static Operations:** The class contains no instance fields and exposes its entire functionality through static methods. This design eliminates object lifecycle management and ensures inherent thread safety at the class level.
2.  **Generic, Functional API:** By leveraging generics and functional interfaces like Consumer and Function, the utility remains decoupled from any specific data type. It can operate on any collection of objects, provided a function is supplied to extract a Vector3d position from each object.

A notable optimization exists for collections implementing the proprietary **FastCollection** interface. When a FastCollection is provided, the utility bypasses the standard Java iterator in favor of a specialized internal iteration mechanism, which is presumed to reduce object allocation and improve cache coherency for performance-critical loops. For all other Iterable types, it falls back to a standard for-each loop.

## Lifecycle & Ownership
- **Creation:** As a static utility class, VectorSphereUtil is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader when it is first referenced by another class.
- **Scope:** The class and its static methods are available for the entire application lifetime once loaded.
- **Destruction:** The class is unloaded if and when its defining ClassLoader is garbage collected, which typically only occurs at application shutdown.

## Internal State & Concurrency
- **State:** VectorSphereUtil is completely stateless. It holds no data between method calls. All necessary information is provided as method arguments, and all results are handled via side effects through the supplied Consumer functions.

- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the operations it performs are **not** guaranteed to be safe. The caller is responsible for ensuring that the input collection and the consumer logic are thread-safe.

    **WARNING:** Passing a non-thread-safe collection (like ArrayList) and modifying it from another thread while forEachVector is executing will result in a ConcurrentModificationException or other undefined behavior. All synchronization must be handled externally.

## API Surface
The API is composed of a series of overloaded methods for spatial iteration and a simple predicate for containment checks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachVector(...) | void | O(N) | Iterates a collection, executing a consumer for each element located inside the defined sphere or ellipsoid. Throws NullPointerException if function or consumer is null. |
| isInside(...) | boolean | O(1) | Returns true if the given Vector3d is within the bounds of the defined sphere or ellipsoid. This is the core mathematical check used by all forEachVector methods. |

## Integration Patterns

### Standard Usage
This utility is intended for filtering moderately sized collections where the overhead of building and maintaining a dedicated spatial index (like an Octree) is not justified. It is ideal for per-tick proximity queries in game systems.

```java
// Find all entities within 10 units of the player's position
List<Entity> allEntities = world.getEntities();
List<Entity> nearbyEntities = new ArrayList<>();
Vector3d playerPosition = player.getPosition();

VectorSphereUtil.forEachVector(
    allEntities,
    Entity::getPosition, // Function to get a Vector3d from an Entity
    playerPosition.getX(),
    playerPosition.getY(),
    playerPosition.getZ(),
    10.0, // Radius
    nearbyEntities::add // Consumer to add matching entities to our list
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and provides no instance-level functionality. Do not attempt to create an instance with `new VectorSphereUtil()`.
- **Large-Scale Broad-Phase Culling:** Using this utility to check millions of objects against each other is inefficient. Its O(N) complexity makes it unsuitable as a replacement for broad-phase culling algorithms that use spatial partitioning structures for O(log N) or O(1) lookups.
- **Concurrent Modification:** Do not modify the input collection within the body of the consumer lambda unless the collection is explicitly designed for concurrent access. This will lead to unpredictable state and runtime exceptions.

## Data Pipeline
VectorSphereUtil acts as a filter and dispatcher within a data flow. It does not transform data but rather selects which data to forward to a subsequent processing step.

> Flow:
> Input Collection -> **VectorSphereUtil.forEachVector** -> (Filter by position) -> Consumer Function -> (Side Effects: e.g., add to new list, apply damage)


---
description: Architectural reference for VectorBoxUtil
---

# VectorBoxUtil

**Package:** com.hypixel.hytale.math.vector
**Type:** Utility

## Definition
```java
// Signature
public class VectorBoxUtil {
```

## Architecture & Concepts
VectorBoxUtil is a high-performance, stateless utility class designed for efficient spatial filtering. Its core responsibility is to iterate over collections of objects, identify which ones reside within a specified axis-aligned bounding box (AABB), and apply a given operation to them.

This class is a foundational component for numerous engine systems that require proximity queries, such as:
*   **Entity Culling:** Determining which entities are within a player's view frustum or a specific region.
*   **AI Systems:** Finding potential targets or points of interest within an AI agent's perception range.
*   **Physics Broad-Phase:** Quickly discarding objects that are too far apart to possibly collide.

The architecture is heavily influenced by functional programming principles, utilizing `Function` and `Consumer` interfaces. This design decouples the spatial query logic from the data structures being queried and the actions being performed, promoting high cohesion and reusability.

A critical architectural feature is the specialized handling for collections that implement the `FastCollection` interface. When a `FastCollection` is provided, VectorBoxUtil bypasses the standard Java `Iterator` mechanism, which can cause performance degradation due to heap allocations in tight game loops. Instead, it leverages a more direct, often zero-allocation iteration path provided by the `FastCollection` implementation, which is essential for maintaining high frame rates.

## Lifecycle & Ownership
- **Creation:** As a static utility class, VectorBoxUtil is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader upon first reference.
- **Scope:** The class and its static methods are available globally throughout the application's lifetime once loaded.
- **Destruction:** The class is unloaded from the JVM when its ClassLoader is garbage collected, which typically occurs only during application shutdown.

## Internal State & Concurrency
- **State:** VectorBoxUtil is entirely **stateless**. It contains no member variables and all its methods are pure functions whose output depends solely on their input arguments.
- **Thread Safety:** The class is inherently **thread-safe**. Since no internal state is maintained, concurrent calls to its methods will not interfere with one another.

    **WARNING:** Thread safety is contingent on the inputs. The caller is responsible for ensuring that the provided collections and lambda expressions are themselves thread-safe. Modifying the input collection from within a consumer lambda during iteration will lead to undefined behavior or a `ConcurrentModificationException` for standard Java collections.

## API Surface
The public API consists of two families of overloaded static methods: `isInside` for point-in-box checks and `forEachVector` for collection filtering.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isInside(...) | boolean | O(1) | Performs an axis-aligned bounding box check on a single Vector3d. |
| forEachVector(...) | void | O(N) | Iterates a collection of N items, applying a consumer to each item found within the defined bounding box. |

## Integration Patterns

### Standard Usage
The primary pattern is to provide a collection, a function to extract a position vector from each element, a bounding box, and a consumer lambda to execute on the filtered elements.

```java
// Example: Find all entities near the player and make them glow.
List<Entity> nearbyEntities = world.getEntities();
Player player = world.getPlayer();
Vector3d playerPos = player.getPosition();

// Define a 10-unit box around the player
double searchRadius = 10.0;

VectorBoxUtil.forEachVector(
    nearbyEntities,
    Entity::getPosition, // Function to get a Vector3d from an Entity
    playerPos.getX(), playerPos.getY(), playerPos.getZ(),
    searchRadius,
    entity -> entity.applyEffect(Effect.GLOWING) // Consumer to apply to found entities
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new VectorBoxUtil()`. This class is not designed to be instantiated and provides no public constructor.
- **Collection Modification:** Do not add or remove elements from the source collection within the consumer lambda. This violates the contract of most Java iterators and will result in a `ConcurrentModificationException` or other unpredictable failures.
- **Ignoring FastCollection:** For performance-critical systems that iterate thousands of times per second (e.g., particle systems), passing a standard `ArrayList` instead of a `FastCollection` implementation will result in significant GC pressure from iterator allocations. Always prefer `FastCollection` where performance is paramount.

## Data Pipeline
VectorBoxUtil acts as a filter and dispatcher within a larger data processing flow. It does not own data but rather operates on data streams passed through it.

> Flow:
> Source Collection (e.g., World Entity List) -> **VectorBoxUtil.forEachVector** (Applies spatial filter) -> Filtered Elements -> Consumer Logic (e.g., AI Behavior Update, Renderer Command)


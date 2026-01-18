---
description: Architectural reference for CaveNodeShapeUtils
---

# CaveNodeShapeUtils

**Package:** com.hypixel.hytale.server.worldgen.cave.shape
**Type:** Utility

## Definition
```java
// Signature
public class CaveNodeShapeUtils {
```

## Architecture & Concepts
CaveNodeShapeUtils is a stateless, static utility class that serves as a core mathematical and logical library for the procedural cave generation system. It centralizes a wide range of complex geometric calculations, shape interpolations, and block placement rules required to carve and decorate cave networks during world generation.

This class is not a service or a stateful manager; it is a collection of pure functions. Its primary architectural purpose is to decouple the high-level cave generation algorithm from the low-level, mathematically-intensive details of shape definition and rule evaluation. By providing a stable, shared toolkit, it ensures that different cave generation components (e.g., those for tunnels, large caverns, or intersections) can perform calculations consistently.

It operates directly on world generation primitives like Vector3d, CaveNode, and the ChunkGeneratorExecution context, acting as a foundational layer for any code that needs to translate abstract cave definitions into concrete block placements within a chunk.

## Lifecycle & Ownership
As a static utility class, CaveNodeShapeUtils does not have a traditional instance lifecycle.

- **Creation:** The class is loaded into the JVM by the class loader when it is first referenced by any part of the server, typically during the startup of the world generation system. It is never instantiated.
- **Scope:** Its scope is global and application-wide. The static methods are available for the entire duration of the server's runtime.
- **Destruction:** The class is unloaded from memory only when the JVM shuts down. There is no concept of instance-based cleanup or resource management.

## Internal State & Concurrency
- **State:** CaveNodeShapeUtils is **entirely stateless and immutable**. It contains no mutable static or instance fields. The only static members are `final` constants, primarily functional interfaces for mathematical operations. All calculations are performed exclusively on the arguments passed into its methods.

- **Thread Safety:** This class is **unconditionally thread-safe**. Its stateless and functional nature guarantees that its methods can be called concurrently from multiple world generation threads without any risk of data corruption or race conditions.

    **WARNING:** While the class itself is thread-safe, the objects passed to it, such as ChunkGeneratorExecution, are typically bound to a single generation thread. Callers are responsible for ensuring that these context objects are not shared or accessed improperly across threads.

## API Surface
The API consists of static methods for geometric calculation and rule evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBoxAnchor(...) | Vector3d | O(1) | Calculates a 3D point within a bounding box, based on fractional coordinates. |
| getSphereAnchor(...) | Vector3d | O(1) | Projects a point onto the surface of a sphere or ellipsoid. |
| getPipeAnchor(...) | Vector3d | O(1) | Calculates a point on the surface of a cylindrical or pipe-like shape. |
| getEndRadius(...) | double | O(1) | Determines the final radius of a cave segment shape, such as a cylinder. |
| getFillingBlock(...) | BlockFluidEntry | O(1) | Determines the appropriate block (e.g., air, water) for a given Y-level based on cave fluid settings. |
| isCoverMatchingParent(...) | boolean | O(1) | Evaluates if a "cover" block (e.g., stalactite) can be placed, based on the properties of the adjacent block. |
| invalidateCover(...) | boolean | O(1) | Checks if an existing block at a target location should prevent a cover block from being placed. |

## Integration Patterns

### Standard Usage
This class is intended to be used directly via static calls from within other world generation components. It is never retrieved from a service locator or dependency injection context.

```java
// Hypothetical usage within a cave decorator
public void decorateCeiling(ChunkGeneratorExecution execution, CaveNode node) {
    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            int y = findCeilingHeight(x, z); // Find the ceiling of the cave

            // Check if a stalactite can be placed here
            boolean canPlace = CaveNodeShapeUtils.isCoverMatchingParent(
                x, z, y, execution, node.getCeilingCover()
            );

            if (canPlace) {
                execution.setBlock(x, y - 1, z, STALACTITE_BLOCK);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create an instance of this class. It is a static utility and will fail. `new CaveNodeShapeUtils()` is invalid.
- **Introducing State:** Do not modify this class to hold static mutable state. Its thread safety and reliability depend on its statelessness. Adding any form of cache or static variable that is modified at runtime is a critical error.
- **Complex Logic:** Avoid adding high-level business logic about specific cave types into this class. It should remain a library of generic, reusable geometric and rule-based functions.

## Data Pipeline
CaveNodeShapeUtils does not participate in a data pipeline in a traditional sense. Instead, it is a functional component invoked *by* a pipeline to perform discrete calculations. It acts as a "calculator" that the main generation loop consults.

A typical flow for cover block placement illustrates its role:

> Flow:
> Cave Generation Algorithm iterates a block position (x, y, z) -> Calls `isCoverMatchingParent` with position and `ChunkGeneratorExecution` -> **CaveNodeShapeUtils** reads the parent block data from the execution context -> The utility evaluates the condition -> Returns a boolean result -> Cave Generation Algorithm uses the result to decide whether to write a new block into the chunk buffer.


---
description: Architectural reference for WorldUtil
---

# WorldUtil

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Utility

## Definition
```java
// Signature
public final class WorldUtil {
```

## Architecture & Concepts
WorldUtil is a low-level, stateless query engine for the server's world representation. It serves as a fundamental bridge between high-level game systems (such as physics, AI, and entity behavior) and the raw, component-based chunk data storage.

Its primary architectural role is to provide a stable, high-performance API that abstracts the complexity of the underlying world data structures. Callers do not need to know the internal layout of a ChunkColumn, BlockSection, or FluidSection; they can simply ask questions about the world at a given coordinate, such as "What is the material here?" or "Where is the next solid surface below this point?".

The class is heavily optimized for vertical lookups (along the Y-axis), a common requirement for gravity, collision, and placement logic. It contains sophisticated logic for handling complex cases, including partial blocks with precise bounding boxes (hitboxes), fluid levels, and special block states like fillers. By centralizing these complex and frequently needed calculations, WorldUtil ensures consistency and performance across the entire server codebase.

## Lifecycle & Ownership
- **Creation:** As a final class with only static methods, WorldUtil is never instantiated. The Java Virtual Machine loads the class into memory at runtime, typically upon first access.
- **Scope:** The class and its static methods are available for the entire lifetime of the server application.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no concept of manual destruction or resource cleanup associated with this utility.

## Internal State & Concurrency
- **State:** WorldUtil is **completely stateless**. It contains no member fields and all operations are performed on data provided via method arguments. Its behavior is deterministic based on its inputs.

- **Thread Safety:** The class itself is inherently **thread-safe** due to its stateless nature. However, the data structures passed into its methods (ComponentAccessor, ChunkColumn, etc.) are not guaranteed to be thread-safe for concurrent modification.

    **WARNING:** Callers are responsible for ensuring that the world data being queried through WorldUtil is not being modified by another thread simultaneously without proper synchronization mechanisms. Failure to do so will result in race conditions and world corruption. It is safest to use this utility from a thread that has exclusive or read-only access to the target chunk data, such as during a world tick.

## API Surface
The API provides direct, efficient methods for querying block, fluid, and material states at specific world coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPackedMaterialAndFluidAtPosition(...) | long | Low | The primary query method. Returns a packed long containing both BlockMaterial and fluid ID for a precise floating-point coordinate. Accounts for hitboxes and fluid levels. |
| findFarthestEmptySpaceBelow(...) | int | O(H) | Performs a vertical raycast downwards to find the Y-coordinate of the first non-empty block. Optimized to skip entire empty chunk sections. |
| findFarthestEmptySpaceAbove(...) | int | O(H) | Performs a vertical raycast upwards to find the Y-coordinate of the first non-empty block. |
| getWaterLevel(...) | int | O(H) | Calculates the surface Y-level of a contiguous body of fluid at a given X/Z column. |
| getFluidIdAtPosition(...) | int | Low | Retrieves the fluid ID at a specific integer block coordinate. |
| isSolidOnlyBlock(...) | boolean | O(1) | A fast utility check to determine if a block is solid and contains no fluid. |

## Integration Patterns

### Standard Usage
WorldUtil should be used by any system that needs to understand the physical composition of the world for its logic. It is almost always used within a system that already has a reference to a specific chunk or set of chunks.

```java
// Example: A physics system determining if an entity is grounded
public void applyGravity(Entity entity, ComponentAccessor<ChunkStore> chunkStore) {
    // Assume we have the entity's current chunk components
    ChunkColumn column = ...;
    BlockChunk blocks = ...;
    int entityX = (int) entity.getX();
    int entityY = (int) entity.getY();
    int entityZ = (int) entity.getZ();

    // Find the ground level directly below the entity
    int groundY = WorldUtil.findFarthestEmptySpaceBelow(
        chunkStore,
        column,
        blocks,
        entityX,
        entityY,
        entityZ,
        -1 // Fail value if no ground is found
    );

    if (entityY <= groundY) {
        entity.setGrounded(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Lookups:** Do not call methods like getPackedMaterialAndFluidAtPosition repeatedly for the same location within a single logic tick. Cache the result in a local variable if it is needed multiple times.
- **Cross-Thread Writes:** Do not query a chunk using WorldUtil on one thread while another thread is actively modifying that same chunk's block data. This will lead to inconsistent results and severe bugs. All world modifications should be synchronized.
- **Ignoring Packed Data:** When using getPackedMaterialAndFluidAtPosition, failing to unpack and check both the material and fluid components can lead to incorrect assumptions. An entity might be inside a non-solid block but still be colliding with a fluid.

## Data Pipeline
WorldUtil functions as a query endpoint in the data flow, not a processing stage. It translates high-level spatial queries into low-level data lookups. The typical flow for its most comprehensive method is as follows:

> Flow:
> Game System (e.g., Physics) -> Provides `(double x, y, z)` coordinates -> **WorldUtil.getPackedMaterialAndFluidAtPosition** -> Accesses `ChunkStore` via `ComponentAccessor` -> Retrieves `ChunkColumn`, `BlockChunk`, `BlockSection`, `FluidSection` -> Reads block ID, fluid ID, filler data, rotation -> Accesses `BlockBoundingBoxes` asset -> Performs precise hitbox and fluid level checks -> Returns packed `long` -> Game System consumes material and fluid data.


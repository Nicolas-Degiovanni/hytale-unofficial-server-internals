---
description: Architectural reference for CollisionModule
---

# CollisionModule

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Singleton

## Definition
```java
// Signature
public class CollisionModule extends JavaPlugin {
```

## Architecture & Concepts
The CollisionModule is a core server plugin responsible for all physics-based collision and intersection detection. It serves as the centralized authority for spatial queries against both the static world geometry (blocks) and dynamic entities.

Architecturally, this module acts as a stateless service with a static API surface for performing complex, stateful operations. The core collision logic is exposed via static methods like findCollisions, which accept a mutable **CollisionResult** object. This parameter object pattern isolates the state of a single collision query from the global module, enabling re-entrant and potentially parallelizable checks.

The module bifurcates its collision detection strategy based on movement distance:
1.  **Short Distance:** For movements below a small threshold, a direct, optimized check (findBlockCollisionsShortDistance) is performed against nearby blocks. This is the common case for player and entity movement.
2.  **Long Distance (Iterative):** For larger movements (e.g., projectiles, teleportation), an iterative ray-marching or sweep-test approach (findBlockCollisionsIterative) is used to ensure no colliders are missed along the movement vector.

For entity-vs-entity collision, the module maintains a spatial partitioning data structure, a **KDTree**, registered as a tangiableEntitySpatialComponent. This allows for efficient spatial lookups of entities, reducing the complexity of character collision checks from O(N^2) to O(N log N).

The module also integrates with the asset pipeline by listening for LoadedAssetsEvent. Upon receiving this event, it processes block hitbox data to pre-calculate and cache global metrics like maximum block extent and minimum hitbox thickness, which are used in subsequent collision optimizations.

## Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's plugin loader during the server bootstrap sequence. The static singleton instance is set within the constructor, making it immediately available via the static get method.
-   **Scope:** The CollisionModule is a global singleton. It persists for the entire lifetime of the server process.
-   **Destruction:** The module is destroyed and its resources are released during server shutdown when the plugin system is dismantled.

## Internal State & Concurrency
-   **State:** The module maintains a small amount of mutable internal state.
    -   **extentMax, minimumThickness:** These are cached floating-point values derived from loaded block assets. They represent global physical properties of the world's blocks.
    -   **tangiableEntitySpatialComponent:** A handle to the server-wide KDTree used for accelerating entity lookups. The tree itself contains the state, not the module.
    -   **config:** A reference to the module's configuration object.

-   **Thread Safety:** This class is **not** thread-safe for mutation but is safe for concurrent reads.
    -   The internal state (extentMax, minimumThickness) is mutated within the onLoadedAssetsEvent handler. This is safe under the assumption that asset loading events are dispatched serially on a main server thread.
    -   The primary public API consists of static methods (e.g., findCollisions). These methods are re-entrant and safe to call from multiple threads, **provided that** each thread uses a separate, non-shared CollisionResult instance. The underlying world data and component accessors must also be safe for concurrent reads.

    **WARNING:** Passing the same CollisionResult object to concurrent calls of findCollisions will result in undefined behavior and data corruption.

## API Surface
The primary API is static, designed to operate on externally provided data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | CollisionModule | O(1) | Returns the singleton instance of the module. |
| findCollisions(...) | boolean | O(log N + M) | The main entry point for collision detection. Sweeps a collider along a vector, populating a CollisionResult. N is the number of entities, M is the number of blocks traversed. |
| findIntersections(...) | void | O(log N + K) | Finds all blocks and entities that currently overlap a given collider at a static position. K is the number of blocks under the collider. |
| validatePosition(...) | int | O(K) | Checks if a collider at a given position is in a valid state (e.g., not overlapping solid blocks). Returns a bitmask indicating status (OK, ON_GROUND, TOUCH_CEIL). |
| getTangiableEntitySpatialComponent() | ResourceType | O(1) | Returns the handle for the spatial resource used to query tangible entities. |

## Integration Patterns

### Standard Usage
The most common use case is to resolve an entity's movement. This involves calling findCollisions with the entity's bounding box, current position, and desired velocity, then using the populated CollisionResult to determine the final, corrected position.

```java
// Standard movement resolution
CollisionResult result = new CollisionResult(); // A new result object is critical
ComponentAccessor<EntityStore> accessor = ...;
Box entityCollider = ...;
Vector3d currentPosition = ...;
Vector3d desiredVelocity = ...;

CollisionModule.findCollisions(entityCollider, currentPosition, desiredVelocity, result, accessor);

// The 'result' object now contains detailed collision data.
// The movement system will use result.getFinalPosition() to update the entity.
Vector3d finalPosition = result.getFinalPosition();
entityTransform.setPosition(finalPosition);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CollisionModule()`. The module is managed by the server's plugin system. Always use `CollisionModule.get()` to retrieve the singleton instance.
-   **Reusing CollisionResult:** Do not reuse a single `CollisionResult` object across multiple, independent collision checks, especially across different threads. This will lead to corrupted and unpredictable outcomes. Always create a new `CollisionResult` for each distinct collision query.
-   **Ignoring the Return Value:** The boolean return value of `findCollisions` indicates whether the long-distance iterative path was taken. Logic that depends on this distinction should check the return value.

## Data Pipeline
The module participates in two primary data flows: asset processing and runtime collision queries.

**1. Asset Processing Pipeline (Server Startup)**
> Flow:
> Block Asset Files -> AssetStore -> **LoadedAssetsEvent** -> CollisionModule::onLoadedAssetsEvent -> **Internal State Cache** (extentMax, minimumThickness)

**2. Runtime Movement Collision Pipeline (Per-Tick)**
> Flow:
> Entity Physics System -> **CollisionModule::findCollisions** -> (Reads from) World Block Storage & Entity KDTree -> **CollisionResult** (Populated) -> Entity Physics System (Applies corrected position)


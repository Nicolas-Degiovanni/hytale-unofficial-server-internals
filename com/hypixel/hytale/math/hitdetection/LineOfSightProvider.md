---
description: Architectural reference for LineOfSightProvider
---

# LineOfSightProvider

**Package:** com.hypixel.hytale.math.hitdetection
**Type:** Functional Interface (Strategy Pattern)

## Definition
```java
// Signature
public interface LineOfSightProvider {
   // ... members
}
```

## Architecture & Concepts
The LineOfSightProvider interface defines a formal contract for ray-based visibility testing within the game world. It serves as a core component of the Strategy Pattern, abstracting the specific algorithm used to determine if a clear path exists between two points. This decoupling is critical for systems like AI, combat mechanics, and rendering culling, allowing them to query for line of sight without being coupled to the underlying world representation (e.g., voxels, meshes).

Different implementations can be provided to model different types of vision or environmental interactions:
*   A standard implementation would raycast against solid world geometry.
*   A specialized implementation might ignore certain materials like glass or foliage.
*   A performance-oriented implementation could use a more coarse-grained acceleration structure.

The interface includes a static constant, DEFAULT_TRUE, which acts as a Null Object. This is a default implementation that always returns true, useful for entities that can see through walls or for disabling visibility checks entirely for performance or gameplay reasons.

## Lifecycle & Ownership
As an interface, LineOfSightProvider itself has no lifecycle. The following pertains to its *implementations*.

-   **Creation:** Implementations are instantiated by various engine systems based on need. For example, the WorldManager may construct a default provider that queries its voxel data. AI systems might create specialized providers for specific agent types. Anonymous implementations are often created on-the-fly as lambdas for single-use checks.
-   **Scope:** The lifetime of a provider implementation is tied to its owner. An AI agent may hold a reference to its provider for its entire existence. The static DEFAULT_TRUE provider is a singleton that persists for the lifetime of the JVM.
-   **Destruction:** Implementations are subject to standard Java garbage collection once they are no longer referenced by any system.

## Internal State & Concurrency
-   **State:** The interface contract strongly implies that implementations should be stateless. The test method's signature, which accepts all necessary data as arguments, suggests it should be a pure function with no side effects. Caching results internally is a possible optimization but introduces complexity regarding cache invalidation and thread safety.
-   **Thread Safety:** Implementations **must be thread-safe**. Line of sight checks are prime candidates for parallelization, especially for AI systems managing numerous agents. Any implementation that is not re-entrant or relies on mutable shared state will introduce severe concurrency bugs. The provided DEFAULT_TRUE is inherently thread-safe as it is stateless and immutable.

## API Surface
The public contract consists of a single functional method and a static default instance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(fromX, fromY, fromZ, toX, toY, toZ) | boolean | Varies | Executes the visibility check. Complexity is implementation-dependent, typically ranging from O(log N) to O(N) based on the raycasting algorithm and world data structure. |
| DEFAULT_TRUE | LineOfSightProvider | O(1) | A static, singleton provider that always returns true. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a central authority, such as the world or an entity's component set, and used to gate logic.

```java
// An AI system checks if it can see its target
Entity target = ...;
LineOfSightProvider sight = world.getLineOfSightProvider();

boolean canSeeTarget = sight.test(
    self.getX(), self.getY(), self.getZ(),
    target.getX(), target.getY(), target.getZ()
);

if (canSeeTarget) {
    // Proceed with attack logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Avoid creating implementations that store state between calls. This breaks the functional contract and is a primary source of concurrency issues when used by multi-threaded systems.
-   **Expensive Blocking Operations:** An implementation of the test method must not perform long-running or blocking operations. Doing so can cause significant stalls in the calling thread, which is often the main game loop or a time-sensitive AI tick.

## Data Pipeline
This component acts as a query function rather than a data pipeline. It takes coordinates as input and produces a boolean result.

> Flow:
> System (e.g., AI) -> **LineOfSightProvider.test(coords)** -> World Geometry Query -> Boolean Result


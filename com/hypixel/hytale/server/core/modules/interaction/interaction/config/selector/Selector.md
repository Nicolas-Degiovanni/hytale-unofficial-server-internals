---
description: Architectural reference for the Selector interface, the core of spatial querying for entities and blocks.
---

# Selector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Utility / Contract

## Definition
```java
// Signature
public interface Selector {
```

## Architecture & Concepts
The Selector interface is a fundamental component of the server's interaction module. It establishes a formal contract for spatial querying and target selection within the game world. Its design serves two primary purposes:

1.  **Stateless Utility Methods:** It provides a suite of static, globally-accessible methods for performing common, immediate spatial queries. These are the workhorses for one-off checks like Area of Effect (AOE) calculations or environmental scans, such as finding all entities or blocks within a given radius.

2.  **Stateful Selection Contract:** It defines an interface for concrete implementations that manage complex, stateful selection logic. An object implementing Selector can maintain a target over time, update its selection logic each tick, and handle more sophisticated targeting behaviors like homing projectiles or persistent AI target acquisition.

This interface acts as a bridge between high-level game logic (e.g., an AI behavior tree, a player ability) and the low-level spatial partitioning data structures managed by the `SpatialResource` system. It abstracts the complexity of querying octrees or spatial grids, providing a clean, intention-driven API.

## Lifecycle & Ownership
As an interface, Selector itself has no lifecycle. However, its static methods and concrete implementations have distinct lifecycle characteristics.

-   **Creation:**
    -   **Static Methods:** Are part of the class definition and are available immediately upon class loading by the JVM. They are stateless and require no instantiation.
    -   **Implementations:** Concrete classes implementing Selector are instantiated by higher-level systems that require persistent targeting logic. For example, an `AbilityComponent` might create a `HomingMissileSelector` when an ability is activated.

-   **Scope:**
    -   **Static Methods:** Global scope. They can be called from anywhere, but are subject to strict threading constraints.
    -   **Implementations:** The scope is tied to the owning object. An implementation will live as long as its parent component or system is active (e.g., for the duration of a spell effect).

-   **Destruction:**
    -   **Static Methods:** Not applicable.
    -   **Implementations:** Marked for garbage collection when the owning object is destroyed or releases its reference. They hold no engine-level resources that require manual cleanup.

## Internal State & Concurrency
-   **State:**
    -   The static methods are **stateless**. They compute their results based solely on the arguments provided.
    -   The interface contract (e.g., the `tick` method) strongly implies that concrete implementations will be **mutable and stateful**, potentially caching target references or tracking cooldowns.

-   **Thread Safety:**
    -   **WARNING:** This component is **NOT thread-safe** for general-purpose use.
    -   The static method `selectNearbyEntities` retrieves a `ThreadLocal` list from `SpatialResource`. This is a critical performance optimization to avoid list allocation on every query.
    -   Consequently, any method that relies on `SpatialResource` **must** be executed on the main server thread responsible for the game tick. Calling these methods from asynchronous tasks or worker threads will lead to exceptions or incorrect behavior. The system is designed to be single-threaded per world tick.

## API Surface
The public contract consists of stateless query functions and a stateful update contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| selectNearbyBlocks(..., range, consumer) | static void | O(R³) | Iterates a cubic volume defined by the range. **Warning:** High performance cost with large ranges. |
| selectNearbyEntities(..., range, consumer, filter) | static void | O(log N + M) | Performs a spherical query against the engine's spatial data structures. N is the total number of entities in the structure; M is the number of entities found. Highly efficient. |
| tick(commandBuffer, ref, deltaTime, totalTime) | void | Varies | Contract for updating the internal state of a selector implementation. Called once per game tick. |
| selectTargetEntities(..., consumer, filter) | void | Varies | Contract for retrieving the current list of selected entities from a stateful implementation. |
| selectTargetBlocks(..., consumer) | void | Varies | Contract for retrieving the current list of selected blocks from a stateful implementation. |

## Integration Patterns

### Standard Usage
The static methods are used for immediate, one-off spatial queries. This is the most common pattern for abilities or environmental interactions.

```java
// Find all entities within 10 units of an attacker that are not the attacker itself.
// This code must be run on the main server thread.

Vector3d center = transform.getPosition();
double range = 10.0;
Predicate<Ref<EntityStore>> filter = (entityRef) -> !entityRef.equals(attackerRef);

Selector.selectNearbyEntities(commandBuffer, center, range, (targetRef) -> {
    // Apply damage or status effect to targetRef
}, filter);
```

## Anti-Patterns (Do NOT do this)
-   **Large-Range Block Selection:** Do not call `selectNearbyBlocks` with a large `range`. The complexity is cubic, and a range of 64 would result in over 1,000,000 iterations (2 * 64)³, causing severe server lag. For large areas, use a more specialized system.
-   **Cross-Thread Access:** Never call `selectNearbyEntities` from a separate thread. The reliance on `ThreadLocal` storage will cause `NullPointerException` or return data from an entirely different query. All Selector usage must be synchronized with the main game loop.

## Data Pipeline
Selector acts as a query interface into the world's spatial data, not as a processing stage in a pipeline. Its primary flow is a request/response cycle initiated by game logic.

> Flow:
> Game Logic (AI, Ability) -> **Selector.selectNearbyEntities** -> SpatialResource (Query) -> ThreadLocal Result List -> **Selector** -> Game Logic (Consumer Callback)


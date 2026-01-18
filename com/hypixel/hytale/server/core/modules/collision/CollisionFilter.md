---
description: Architectural reference for CollisionFilter
---

# CollisionFilter

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface CollisionFilter<D, T> {
   boolean test(T var1, int var2, D var3, CollisionConfig var4);
}
```

## Architecture & Concepts
The CollisionFilter interface defines a behavioral contract for fine-grained collision culling within the server's physics and collision detection module. It embodies the **Strategy Pattern**, allowing the core collision engine to remain agnostic of game-specific logic while providing developers a hook to implement custom interaction rules.

Its primary function is to act as a predicate, or a gate, during the narrowphase of collision detection. After two potential colliders are identified by the broadphase algorithm, the engine invokes an associated CollisionFilter. The filter's return value determines whether the collision should be processed further (resolution and event dispatch) or ignored entirely.

This mechanism is critical for performance and gameplay implementation. It enables scenarios such as:
- Allowing projectiles to pass through teammates but not enemies.
- Making certain entities "ethereal" or non-collidable under specific conditions.
- Implementing complex collision layers that go beyond simple bitmasking.

The generic types, `T` and `D`, represent the two potentially colliding objects, providing type-safe access to their data within the filter's implementation.

## Lifecycle & Ownership
As a functional interface, CollisionFilter itself does not have a lifecycle. Instead, the lifecycle pertains to its concrete implementations (typically lambdas or anonymous classes).

- **Creation:** Implementations are created and supplied by game logic code. They are typically instantiated when defining an entity's physical properties or when registering a specific collision interaction pair with the physics world.
- **Scope:** The lifetime of a CollisionFilter implementation is bound to the object that holds a reference to it. This is most often a CollisionConfig instance, an entity's physics component, or a global physics world setting.
- **Destruction:** The filter instance is eligible for garbage collection when its owning object is destroyed or when the reference to it is cleared.

## Internal State & Concurrency
- **State:** Implementations of CollisionFilter **must be stateless**. They are predicates designed to make a decision based solely on the provided arguments. Introducing mutable internal state can lead to non-deterministic physics simulations, difficult-to-trace bugs, and concurrency issues. The filter should be a pure function where possible.

- **Thread Safety:** **CRITICAL:** The server's physics simulation may be executed across multiple threads to improve performance. Therefore, all implementations of CollisionFilter **must be unconditionally thread-safe**. Given the requirement to be stateless, this is typically achieved by default. Avoid accessing any shared, mutable state from within the filter's implementation without proper synchronization, as this will lead to severe race conditions.

## API Surface
The interface exposes a single method that forms its entire public contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T, int, D, CollisionConfig) | boolean | O(1) | Executes the collision predicate. Returns **true** if the collision should be processed, **false** if it should be ignored. |

**Parameter Breakdown:**
- **T var1:** The first entity or shape involved in the potential collision.
- **int var2:** An integer identifier, often representing a specific shape index within a compound collider for `var1`.
- **D var3:** The second entity or shape involved in the potential collision.
- **CollisionConfig var4:** The active collision configuration context, providing access to global rules, masks, or other system-level data.

## Integration Patterns

### Standard Usage
The intended use is to provide a lambda expression to a system that configures physics interactions. This provides a clean, inline definition of game-specific collision logic.

```java
// Example: Configuring a projectile to ignore its owner
CollisionSystem collisionSystem = context.getService(CollisionSystem.class);
Entity owner = ...;
Entity projectile = ...;

CollisionConfig config = new CollisionConfig();
config.setFilter((entityA, shapeIndex, entityB, cfg) -> {
    // Ignore collision if entityB is the projectile's owner
    if (entityB.getId() == owner.getId()) {
        return false; // Do not collide
    }
    return true; // Collide in all other cases
});

collisionSystem.setCollisionConfig(projectile, config);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create filters that rely on internal or external mutable state. This breaks determinism and thread safety.

    ```java
    // ANTI-PATTERN: Filter relies on an external, mutable flag
    boolean isPhased = false; // This flag can be changed by another thread
    
    config.setFilter((a, i, b, c) -> {
        return !isPhased; // Result is non-deterministic and not thread-safe
    });
    ```

- **Expensive Computations:** The `test` method is in the hot path of the physics loop. Do not perform file I/O, network requests, database queries, or other computationally expensive operations within the filter. This will cause catastrophic performance degradation of the server.

- **Ignoring Parameters:** Do not write a filter that ignores its input parameters. The context provided to the `test` method is essential for making a correct decision.

## Data Pipeline
The CollisionFilter acts as a conditional gate within the broader collision detection pipeline.

> Flow:
> Broadphase (AABB Check) -> Narrowphase (Precise Shape Test) -> **CollisionFilter.test()** -> [If True] -> Collision Resolution (Apply forces) -> Event Bus (Dispatch CollisionEvent)


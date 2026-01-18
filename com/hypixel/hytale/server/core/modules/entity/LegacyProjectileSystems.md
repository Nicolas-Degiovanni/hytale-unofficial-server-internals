---
description: Architectural reference for LegacyProjectileSystems
---

# LegacyProjectileSystems

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Utility

## Definition
```java
// Signature
public class LegacyProjectileSystems {
    // Nested static classes: OnAddHolderSystem, OnAddRefSystem, TickingSystem
}
```

## Architecture & Concepts
The **LegacyProjectileSystems** class is a non-instantiable container that groups a collection of related systems responsible for the entire lifecycle of projectile entities within the server's Entity Component System (ECS). It is not a single system itself, but rather a logical module that enforces a clear separation of concerns for projectile management.

This class embodies a phased approach to entity lifecycle management, broken into three distinct stages, each handled by a dedicated nested class:

1.  **OnAddHolderSystem (Initialization):** This system is the first to act upon the creation of an entity with a **ProjectileComponent**. Its sole responsibility is to bootstrap the entity by adding essential components required for its existence in the world, such as a **NetworkId** for client synchronization, a **ModelComponent** for rendering, and a **BoundingBox** for physics calculations.

2.  **OnAddRefSystem (Validation):** Immediately following initialization, this system performs a critical validation check. It ensures the projectile has been configured correctly, primarily by verifying the presence of a valid projectile asset. If validation fails, this system is responsible for immediately queuing the entity for removal, preventing malformed or "zombie" projectiles from entering the game loop.

3.  **TickingSystem (Behavior & Update):** This is the primary workhorse system that executes every server tick for all active projectile entities. It manages the ongoing behavior, including applying physics updates via the **SimplePhysicsProvider**, decrementing the projectile's lifetime timer, and handling the projectile's death sequence.

This architectural pattern ensures that projectiles are robustly initialized, validated, and updated in a predictable and isolated manner, preventing cascading failures and simplifying debugging.

### Lifecycle & Ownership
The systems defined within **LegacyProjectileSystems** are not transient game objects. They are foundational, long-lived services managed by the core ECS framework.

-   **Creation:** Instances of **OnAddHolderSystem**, **OnAddRefSystem**, and **TickingSystem** are created once by the server's central ECS orchestrator during the server bootstrap or module loading phase. They are never instantiated directly by game logic.
-   **Scope:** These system instances are singletons within the context of the ECS. They persist for the entire duration of the server session.
-   **Destruction:** The systems are discarded only during a full server shutdown when the ECS itself is torn down.

## Internal State & Concurrency
-   **State:** The **LegacyProjectileSystems** container is stateless. The nested systems are designed to be stateless as well, a core principle of ECS. They hold immutable references to **ComponentType** definitions for efficient component access but do not store per-entity data. All mutable state is stored within the components attached to the entities they process.

-   **Thread Safety:** These systems are **not thread-safe** and must be executed by a single-threaded, deterministic game loop. All operations that modify entity state (e.g., adding components, removing entities) are funneled through a **CommandBuffer**. This pattern defers state changes to a safe synchronization point at the end of the tick, preventing race conditions and state corruption. The **TickingSystem**'s implementation of the **DisableProcessingAssert** interface is a critical indicator that it operates under specific, and potentially relaxed, concurrency assertions, likely for performance reasons.

    **WARNING:** Calling any system methods from an external thread will lead to severe state corruption and server instability.

## API Surface
The public API is not meant for direct invocation but rather for integration with the ECS framework. The key entry points are the overridden methods from the base system classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OnAddHolderSystem.onEntityAdd | void | O(1) | Initializes a new projectile entity. Adds **NetworkId**, **ModelComponent**, and **BoundingBox**. |
| OnAddRefSystem.onEntityAdded | void | O(1) | Validates a new projectile. Queues removal via **CommandBuffer** if initialization failed. |
| TickingSystem.tick | void | O(1) | Executes per-tick logic for a single projectile. Manages lifetime and physics. |

## Integration Patterns

### Standard Usage
A developer does not interact with these systems directly. The systems are automatically invoked by the ECS framework. To create and manage a projectile, one simply creates an entity and adds the **ProjectileComponent** to it. The systems will then handle the rest of the lifecycle automatically.

```java
// Correctly creating a projectile entity which these systems will manage.
// This code would exist within another system or entity creation logic.

// 1. Create the entity within a CommandBuffer.
Ref<EntityStore> projectileEntity = commandBuffer.createEntity();

// 2. Add the core ProjectileComponent with its configuration.
// The LegacyProjectileSystems will automatically detect this entity.
commandBuffer.addComponent(projectileEntity, new ProjectileComponent(...));

// 3. Add required components for the TickingSystem to operate.
commandBuffer.addComponent(projectileEntity, new TransformComponent(...));
commandBuffer.addComponent(projectileEntity, new Velocity(...));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LegacyProjectileSystems.TickingSystem(...)`. The ECS framework is responsible for creating and managing system instances. Direct creation will result in a system that is not registered and will never be executed.
-   **Manual Invocation:** Do not call the **tick** or **onEntityAdd** methods manually. This bypasses the **CommandBuffer** and the ECS scheduler, which will cause immediate state corruption, concurrency violations, and unpredictable crashes.
-   **Component Omission:** Creating a projectile entity without a **TransformComponent** or **Velocity** will cause the **TickingSystem** to fail its query, and the projectile will remain static and never update.

## Data Pipeline
The flow of data and control for a projectile entity is strictly managed by the sequence of systems within this class.

> **Creation & Validation Flow:**
> Game Logic creates an entity with **ProjectileComponent** -> ECS dispatches to **OnAddHolderSystem** -> Adds **NetworkId**, **ModelComponent**, **BoundingBox** -> ECS dispatches to **OnAddRefSystem** -> Validates configuration -> If invalid, queues entity for removal.

> **Per-Tick Update Flow:**
> Game Loop Tick -> ECS queries for entities matching the **TickingSystem** archetype -> **TickingSystem.tick** is called for each projectile -> Reads **TransformComponent**, updates **Velocity**, checks lifetime timer -> Queues state changes (e.g., entity removal) to the **CommandBuffer**.


---
description: Architectural reference for PhysicsValuesAddSystem
---

# PhysicsValuesAddSystem

**Package:** com.hypixel.hytale.server.core.modules.physics.systems
**Type:** System Component

## Definition
```java
// Signature
public class PhysicsValuesAddSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The PhysicsValuesAddSystem is a reactive, event-driven system operating within the server's Entity Component System (ECS) framework. Its sole responsibility is to **bootstrap the physics state for newly created entities**.

This system functions as a component initializer. It subscribes to a specific query that targets entities which have been created but do not yet possess a PhysicsValues component. Upon detecting such an entity, it performs two critical actions:
1.  It adds a new, default-initialized PhysicsValues component to the entity.
2.  It attempts to locate a ModelComponent on the same entity. If found, it copies the predefined physics properties from the entity's model data into the newly created PhysicsValues component.

This design decouples entity creation logic from physics initialization. Game logic can spawn an entity with just a model, and this system guarantees that the necessary physics component will be attached automatically in a subsequent, deterministic step of the game tick.

The explicit dependency ordering, which forces this system to run *after* model-related systems, is a critical architectural feature. It prevents race conditions where this system might execute before an entity's model has been fully attached and configured.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central `SystemRegistry` during the bootstrap phase of a world instance. It is not intended for manual creation by gameplay programmers.
-   **Scope:** The lifecycle of a PhysicsValuesAddSystem instance is bound to the lifecycle of the `EntityStore` (the world) it services. It persists for the entire server session of that world.
-   **Destruction:** The instance is marked for garbage collection when the server world is shut down and the managing `SystemRegistry` is cleared.

## Internal State & Concurrency
-   **State:** This system is effectively stateless. Its internal fields, such as the component type reference and the entity query, are configured at construction and are immutable thereafter. It does not cache entity data or maintain any state between ticks. All state modification is performed directly on the entity components passed to it by the ECS framework.
-   **Thread Safety:** This system is **not thread-safe** and must only be executed by the main server thread that manages the ECS update cycle. The ECS framework guarantees this single-threaded execution model. Any attempt to invoke its methods from an asynchronous task or external thread will lead to world state corruption and severe concurrency violations.

## API Surface
The public methods of this class are callbacks designed for invocation by the ECS framework, not for direct use by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Triggered when a new entity matches the system's query. This is the primary entry point for the system's logic. |
| getDependencies() | Set | O(1) | Framework callback. Returns the set of systems that must execute before this one, ensuring deterministic ordering. |
| getQuery() | Query | O(1) | Framework callback. Returns the query used by the ECS framework to identify which entities this system should operate on. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Its operation is transparent. The standard usage pattern is to create an entity with a ModelComponent; the framework will then automatically invoke this system to add the corresponding PhysicsValues.

```java
// CONCEPTUAL EXAMPLE:
// This code does NOT call PhysicsValuesAddSystem directly.
// Its execution is an implicit side-effect of creating the entity.

// 1. A system or spawner creates a new entity.
Holder<EntityStore> newEntity = world.createEntity();

// 2. A ModelComponent is added. The entity now matches the query for
//    PhysicsValuesAddSystem because it has a model but no physics.
newEntity.addComponent(ModelComponent.class, component -> {
    component.setModel(modelRegistry.get("boar"));
});

// 3. Later in the same tick, the ECS framework runs PhysicsValuesAddSystem.
// 4. The system automatically adds and configures the PhysicsValues component.
//    The entity is now ready for the physics simulation.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PhysicsValuesAddSystem()`. The system's lifecycle is exclusively managed by the server's `SystemRegistry` to ensure proper dependency injection and execution ordering.
-   **Manual Component Addition:** While possible, manually adding a default PhysicsValues component during entity creation is redundant. This system is designed to centralize and automate this logic. Doing so manually can lead to inconsistent initialization if the logic ever changes.
-   **Dependency Misconfiguration:** Creating another system that reads or modifies the PhysicsValues component without declaring an `Order.AFTER` dependency on PhysicsValuesAddSystem can introduce subtle race conditions. Your system might execute before this one has had a chance to add the component, resulting in null references or unexpected behavior.

## Data Pipeline
The system acts as a simple but critical step in the entity initialization pipeline. It transforms an entity from a purely visual object into one that is ready for physical simulation.

> Flow:
> Entity Creation Event -> ECS Framework matches entity against Query -> **PhysicsValuesAddSystem.onEntityAdd** -> Reads ModelComponent Data -> Creates & Populates PhysicsValues Component -> Entity is now physics-ready.


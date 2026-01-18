---
description: Architectural reference for Projectile
---

# Projectile

**Package:** com.hypixel.hytale.server.core.modules.projectile.component
**Type:** Singleton

## Definition
```java
// Signature
public class Projectile implements Component<EntityStore> {
```

## Architecture & Concepts
The Projectile class is a **marker component** within the server's Entity-Component-System (ECS) architecture. It contains no data or logic; its sole purpose is to be attached to an entity to classify it as a projectile. This design pattern is highly efficient, allowing systems to quickly query for all projectile entities without inspecting their internal state.

This component is implemented as a stateless singleton. A single, static INSTANCE is shared across all entities that are marked as projectiles. This avoids the memory overhead of creating a new Projectile object for every arrow, fireball, or other projectile in the world.

The actual projectile behavior, such as movement, collision detection, and lifespan management, is handled by a dedicated **ProjectileSystem**. That system operates on the set of entities that possess this Projectile component. The component's existence on an entity effectively subscribes that entity to the logic of the ProjectileSystem.

## Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated eagerly by the JVM when the Projectile class is first loaded. It is not created on a per-entity basis.
- **Scope:** The INSTANCE is static and persists for the entire lifetime of the server process.
- **Destruction:** The INSTANCE is garbage collected only when the server application shuts down and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** The Projectile component is **stateless and immutable**. It has no member fields and its identity never changes.
- **Thread Safety:** This class is inherently thread-safe. The single, immutable instance can be safely referenced and attached to entities from any thread without requiring locks or other synchronization primitives.

## API Surface
The public contract is minimal, focusing on providing access to the singleton instance and its type information for the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | Projectile | O(1) | Provides global access to the single, shared instance of the component. |
| CODEC | BuilderCodec | O(1) | A codec for serialization. When deserializing, it is optimized to always return the static INSTANCE rather than creating a new object. |
| getComponentType() | ComponentType | O(1) | Retrieves the unique component type identifier from the central ProjectileModule. This is the primary mechanism for ECS queries. |
| clone() | Component | O(1) | Returns the singleton instance itself. This is an optimization that prevents object allocation when an entity's components are duplicated. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. It is managed by higher-level entity creation systems or APIs. The component is attached to an entity to grant it projectile behaviors.

```java
// Conceptual example of adding the component to an entity
// In practice, this is handled by entity factories or archetypes.

Entity myArrow = world.createEntity();
ComponentType<EntityStore, Projectile> type = Projectile.getComponentType();

// Attaches the shared instance to the entity
myArrow.addComponent(type, Projectile.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not modify this class to include fields (e.g., velocity, owner). Such data belongs in separate components (e.g., a Velocity component, an Ownership component). Adding state would break the stateless singleton pattern and violate ECS principles.
- **Direct Comparison:** Avoid checking if an entity's component is the same instance as Projectile.INSTANCE. The correct ECS pattern is to check for the *presence* of the component type.
    - **Bad:** `if (entity.getComponent(Projectile.class) == Projectile.INSTANCE)`
    - **Good:** `if (entity.hasComponent(Projectile.getComponentType()))`

## Data Pipeline
The Projectile component does not process data. Instead, it acts as a tag that directs an entity into a specific data processing pipeline managed by other systems.

> Flow:
> Entity Archetype Definition -> Entity Instantiation -> **Projectile Component Attached** -> ProjectileSystem (queries for entities with this component) -> Physics Update -> Collision Event Generation


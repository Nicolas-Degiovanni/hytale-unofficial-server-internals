---
description: Architectural reference for RespondToHit
---

# RespondToHit

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class RespondToHit implements Component<EntityStore> {
```

## Architecture & Concepts
The RespondToHit class is a **marker component** within the server-side Entity-Component-System (ECS) architecture. It is a stateless singleton whose sole purpose is to "tag" an entity, signifying that it is capable of reacting to being struck in the game world.

This component holds no data. Its presence on an entity is the data itself. Systems, such as a hypothetical CombatSystem, will query for entities that possess the RespondToHit component to trigger damage calculations, knockback effects, or visual feedback. It acts as a filter, separating entities that are inert (like a rock) from those that are interactive (like a creature or a destructible barrel).

The use of a singleton instance is a critical optimization. Since the component is stateless, every entity that can respond to a hit shares the exact same Java object instance, dramatically reducing memory overhead compared to instantiating a new object for every entity.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated once by the Java Virtual Machine during class loading. It is not created on a per-entity basis.
- **Scope:** The component exists for the entire lifetime of the server process. It is a global, static object.
- **Destruction:** The object is garbage collected only when the server application shuts down and its class loader is unloaded.

## Internal State & Concurrency
- **State:** This component is **immutable and stateless**. It contains no instance fields and its behavior never changes.
- **Thread Safety:** This class is inherently thread-safe. The single, immutable instance can be safely referenced and read by any number of systems across multiple threads without requiring any locks or synchronization primitives.

## API Surface
The public API is minimal, primarily for framework integration rather than direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component from the EntityModule. |
| clone() | Component | O(1) | Returns the shared singleton INSTANCE. This override ensures no new objects are ever created. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its presence is typically defined in entity data files (e.g., prefabs) or added by a system. Other systems then query for entities with this component to apply game logic.

```java
// A hypothetical system processing a hit event
void processHit(Entity attacker, Entity victim) {
    // Check if the victim is tagged with the RespondToHit component
    if (victim.hasComponent(RespondToHit.getComponentType())) {
        // Apply damage, knockback, etc.
        HealthComponent health = victim.getComponent(HealthComponent.class);
        if (health != null) {
            health.takeDamage(10);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to create an instance via reflection would violate the design contract and bypass engine optimizations.
- **Storing Hit Data:** Do not attempt to modify this component to store data about a specific hit (e.g., damage amount, direction). That is the responsibility of an **event** or a separate, stateful component. This class is purely a tag.

## Data Pipeline
This component does not process data; it is a flag used by other systems to route data. It enables an entity to be part of a specific data processing path.

> Flow:
> Collision Event -> CombatSystem -> Query for **RespondToHit** component on entity -> If present, apply damage via HealthComponent -> Trigger AnimationSystem


---
description: Architectural reference for PhysicsValues
---

# PhysicsValues

**Package:** com.hypixel.hytale.server.core.modules.physics.component
**Type:** Data Component

## Definition
```java
// Signature
public class PhysicsValues implements Component<EntityStore> {
```

## Architecture & Concepts
The PhysicsValues class is a data-only component within the server's Entity-Component-System (ECS) architecture. It does not contain any logic. Instead, it serves as a plain data container that attaches fundamental physical properties—such as mass and drag—to an entity.

This component is the primary data source for the server's **Physics System**. The Physics System queries the game world for all entities that possess a PhysicsValues component and uses its data to perform simulation calculations like applying gravity, calculating momentum, and resolving collisions.

A critical feature is the static CODEC field. This defines the data contract for serialization and deserialization, allowing game designers to define and tune an entity's physical behavior in external data files (e.g., JSON or HOCON) without modifying engine code. This data-driven approach is central to the engine's design philosophy.

## Lifecycle & Ownership
- **Creation:** PhysicsValues instances are not meant to be instantiated directly by game logic. They are created by the entity management framework, typically when an entity is spawned from a predefined template. The framework uses the component's CODEC to deserialize properties from the entity template file.
- **Scope:** The lifecycle of a PhysicsValues instance is strictly bound to the lifecycle of the entity it is attached to. It exists only as long as its parent entity is active in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when the component is explicitly removed from the entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. Fields like mass and dragCoefficient are designed to be read and modified at runtime by various game systems (e.g., a magic spell that temporarily reduces an entity's mass). The state is self-contained and holds no references to other engine objects.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All reads and writes must be externally synchronized. Within the Hytale engine, it is assumed that all components belonging to a single entity are processed by a single, designated thread (e.g., the main server tick thread) during any given update cycle to prevent data races.

**WARNING:** Unsynchronized, concurrent access from multiple threads will lead to simulation instability and non-deterministic behavior.

## API Surface
The public API is designed for simple data manipulation and retrieval. Getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique identifier for this component type from the EntityModule registry. |
| replaceValues(other) | void | O(1) | Overwrites all internal fields with values from another PhysicsValues instance. |
| resetToDefault() | void | O(1) | Reverts the component's state to the hardcoded default values. |
| scale(scale) | void | O(1) | Multiplies the mass and dragCoefficient by a given scalar. Useful for dynamic resizing effects. |
| clone() | Component | O(1) | Creates a deep copy of the component, returning a new instance with identical values. |

## Integration Patterns

### Standard Usage
A system, such as a custom gameplay logic system, would retrieve this component from an entity to read or modify its physical properties.

```java
// Example of a system modifying an entity's gravity
void applyGravityWellEffect(Entity targetEntity) {
    // Retrieve the component from the entity. Never create it directly.
    PhysicsValues physics = targetEntity.getComponent(PhysicsValues.getComponentType());

    if (physics != null) {
        // Modify the component's state. The Physics System will use this
        // new value in its next simulation step.
        physics.scale(2.0f); // Double the entity's mass
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PhysicsValues()`. Components must be added to entities via the `EntityManager` or a similar factory, which ensures they are correctly registered and initialized.
- **State Caching:** Do not read a value from this component and store it in another system for later use. The component's state can be changed by other systems at any time. Always query the component directly to ensure you have the most up-to-date data.
- **Unsynchronized Modification:** Do not modify a PhysicsValues component from an asynchronous task (e.g., a network packet handler) without proper synchronization or queuing the modification to run on the main game thread.

## Data Pipeline
PhysicsValues acts as a data source for the physics simulation pipeline. It does not process data itself.

> Flow:
> Entity Template File (JSON/HOCON) -> `CODEC` Deserializer -> **PhysicsValues** (Instance on Entity) -> Physics System (Read) -> World Simulation State Update


---
description: Architectural reference for ForceProviderEntity
---

# ForceProviderEntity

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class ForceProviderEntity extends ForceProviderStandard {
```

## Architecture & Concepts
The ForceProviderEntity class is a concrete implementation of the ForceProviderStandard, designed to supply physical properties for a game entity to the physics simulation engine. It acts as an adapter, translating the geometric properties of an entity's BoundingBox into physical metrics like mass, volume, and projected area.

This class is fundamentally tied to a specific entity instance via its BoundingBox. The physics engine queries this provider during simulation ticks to understand how the entity should react to environmental forces such as fluid dynamics (drag, buoyancy). For example, when calculating air resistance, the engine calls getProjectedArea to determine the entity's cross-sectional area relative to its velocity vector.

**WARNING:** This class is marked as **Deprecated**. It represents an older design pattern for coupling entity state with the physics system. New development should use the recommended component-based architecture, which offers better separation of concerns and data-driven configuration. Continued use of this class may lead to incompatibility with future engine updates.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, `new ForceProviderEntity(boundingBox)`. This is typically done by a higher-level system responsible for constructing the physics representation of a game entity when it is spawned into the world.
- **Scope:** The lifecycle of a ForceProviderEntity is tightly coupled to the entity it represents. It is intended to exist for as long as the entity is actively part of the physics simulation.
- **Destruction:** The object has no explicit destruction or cleanup methods. It is a plain Java object subject to garbage collection once it is no longer referenced by the physics engine or the entity's physics component.

## Internal State & Concurrency
- **State:** This class is highly mutable. It maintains references to a BoundingBox and a ForceProviderStandardState, both of which can be changed after instantiation. The internal density field is also mutable. This design allows for dynamic changes to an entity's physical properties at runtime (e.g., changing material density).
- **Thread Safety:** **This class is not thread-safe.** All fields are accessed and modified without any synchronization mechanisms. All interactions with an instance of this class must be strictly confined to the single thread that runs the physics simulation. Unsynchronized access from other threads will lead to race conditions, inconsistent physics calculations, and engine instability.

## API Surface
The public API is designed to fulfill the contract required by the physics simulation engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setDensity(double) | void | O(1) | Overwrites the material density used for mass calculations. |
| setForceProviderStandardState(state) | void | O(1) | Sets the state object containing environmental data. **CRITICAL:** Failure to set this will result in a NullPointerException. |
| getMass(double) | double | O(1) | Calculates the entity's mass based on its volume and density. |
| getVolume() | double | O(1) | Retrieves the entity's volume directly from its BoundingBox. |
| getProjectedArea(bodyState, speed) | double | O(N) | Calculates the 2D cross-sectional area of the entity relative to its velocity. Complexity depends on the underlying geometry calculations in PhysicsMath. |

## Integration Patterns

### Standard Usage
This pattern is obsolete and provided for reference only. The provider is instantiated and configured by the entity's physics controller, then used by the physics engine.

```java
// This usage is DEPRECATED.
// Assume 'entity' has a BoundingBox component.
BoundingBox entityBounds = entity.getComponent(BoundingBox.class);
ForceProviderEntity physicsProvider = new ForceProviderEntity(entityBounds);

// Configure dynamic properties
physicsProvider.setDensity(850.0); // e.g., Oak wood

// The physics body would then hold a reference to this provider.
PhysicsBody body = new PhysicsBody(physicsProvider);
physicsEngine.addBody(body);
```

### Anti-Patterns (Do NOT do this)
- **Use in New Code:** The primary anti-pattern is using this class at all. It is deprecated and will be removed. Use the modern, component-based alternatives.
- **State Omission:** Instantiating the class but failing to call setForceProviderStandardState before it is used by the physics engine will cause a NullPointerException when getForceProviderStandardState is invoked.
- **Cross-Thread Modification:** Modifying the density or BoundingBox from a thread other than the main physics thread is a severe concurrency violation that will corrupt the simulation state.

## Data Pipeline
ForceProviderEntity acts as a data source, not a processing node. It responds to synchronous queries from the physics engine.

> Flow:
> Physics Engine Tick -> Query for Body Properties -> **ForceProviderEntity.getMass() / getProjectedArea()** -> Read from BoundingBox -> Return Calculated Value -> Physics Engine Force Calculation


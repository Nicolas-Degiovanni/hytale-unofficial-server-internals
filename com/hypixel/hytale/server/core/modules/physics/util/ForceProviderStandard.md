---
description: Architectural reference for ForceProviderStandard
---

# ForceProviderStandard

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility / Base Class

## Definition
```java
// Signature
public abstract class ForceProviderStandard implements ForceProvider {
```

## Architecture & Concepts
ForceProviderStandard is an abstract base class that provides a foundational implementation for calculating common physical forces within the server-side physics engine. It serves as a template for concrete force providers that are attached to physical entities.

This class centralizes the logic for calculating forces such as gravity, air drag, ground friction, and buoyancy. It operates on a Strategy and Template Method pattern: the `update` method is the fixed template algorithm, while the abstract methods (e.g., getMass, getVolume, getFrictionCoefficient) must be implemented by subclasses to provide the specific physical properties of an entity.

This design decouples the physics simulation loop from the specific characteristics of different entity types. The physics engine can treat all entities uniformly through the ForceProvider interface, while subclasses of ForceProviderStandard define the unique physical behaviors of players, items, or other dynamic objects.

### Lifecycle & Ownership
- **Creation:** This class is abstract and cannot be instantiated directly. Concrete subclasses are instantiated and owned by a corresponding PhysicsBody or the entity that the PhysicsBody represents. Creation typically occurs when an entity is spawned into the world.
- **Scope:** The lifecycle of a ForceProviderStandard instance is tightly coupled to its owning PhysicsBody. It persists for the entire duration that the entity is active and being simulated in the game world.
- **Destruction:** The object is eligible for garbage collection when its owning PhysicsBody is destroyed and removed from the physics simulation.

## Internal State & Concurrency
- **State:** The class maintains a minimal, mutable internal state. The primary stateful member is the `dragForce` Vector3d, which is reused across `update` calls to prevent frequent memory allocations. This is a performance optimization common in game engine development. All other physical parameters are retrieved on-demand from subclasses or the associated ForceProviderStandardState.

- **Thread Safety:** This class is **not thread-safe**. The `update` method modifies the internal `dragForce` vector and the passed-in ForceAccumulator. It is designed to be executed exclusively within a single, dedicated physics thread as part of the server's main game loop. Concurrent access from multiple threads will lead to race conditions and unpredictable physics behavior.

## API Surface
The public contract is defined by the `update` method and the abstract methods that subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(bodyState, accumulator, onGround) | void | O(1) | Core method. Calculates and applies standard forces to the ForceAccumulator. |
| clipForce(value, threshold) | void | O(1) | Utility to clamp a force vector within a given threshold. |
| getMass(volume) | abstract double | O(1) | **Subclass Responsibility.** Must return the entity's mass. |
| getVolume() | abstract double | O(1) | **Subclass Responsibility.** Must return the entity's volume. |
| getDensity() | abstract double | O(1) | **Subclass Responsibility.** Must return the entity's density. |
| getProjectedArea(state, speed) | abstract double | O(1) | **Subclass Responsibility.** Must return the cross-sectional area for drag calculations. |
| getFrictionCoefficient() | abstract double | O(1) | **Subclass Responsibility.** Must return the coefficient of friction with the ground. |
| getForceProviderStandardState() | abstract ForceProviderStandardState | O(1) | **Subclass Responsibility.** Must return the state object containing gravity, etc. |

## Integration Patterns

### Standard Usage
A concrete implementation is created and associated with a PhysicsBody. The physics engine then calls the `update` method on each simulation tick.

```java
// Example of a concrete implementation for a Player entity
public class PlayerForceProvider extends ForceProviderStandard {
    private final Player player;
    private final ForceProviderStandardState standardState;

    // Constructor...

    @Override
    public double getMass(double volume) {
        return this.player.getMass();
    }

    // ... other required implementations ...
}

// In the physics engine loop:
PhysicsBody body = player.getPhysicsBody();
ForceAccumulator accumulator = body.getForceAccumulator();
ForceProvider provider = body.getForceProvider(); // This is an instance of PlayerForceProvider

// The engine calls update, which executes the logic in ForceProviderStandard
provider.update(body.getState(), accumulator, body.isOnGround());
```

### Anti-Patterns (Do NOT do this)
- **Stateful Subclasses:** Avoid adding complex, mutable state to subclasses that persists between `update` calls. The provider should be a stateless calculator, deriving its inputs from the PhysicsBodyState and external configuration.
- **Concurrent Modification:** Never call the `update` method from any thread other than the main physics simulation thread. Do not access or modify the provider while the physics engine is ticking.
- **Misusing clipForce:** The `clipForce` method is a low-level utility. Do not call it on the final ForceAccumulator; it is intended for intermediate force calculations like drag.

## Data Pipeline
This class is a core component in the force calculation stage of the physics simulation pipeline.

> Flow:
> Physics Engine Tick -> **ForceProviderStandard.update()** [calculates gravity, drag, friction] -> ForceAccumulator [modified in-place] -> Physics Integrator [calculates new velocity and position] -> Updated PhysicsBodyState

---


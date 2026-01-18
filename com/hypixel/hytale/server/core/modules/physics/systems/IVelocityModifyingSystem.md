---
description: Architectural reference for IVelocityModifyingSystem
---

# IVelocityModifyingSystem

**Package:** com.hypixel.hytale.server.core.modules.physics.systems
**Type:** Marker Interface

## Definition
```java
// Signature
public interface IVelocityModifyingSystem extends ISystem<EntityStore> {
}
```

## Architecture & Concepts
IVelocityModifyingSystem is a **marker interface** that establishes a formal contract for systems participating in the server-side physics simulation. It does not define any methods of its own; its sole purpose is to categorize a class as a "Velocity Modifier". By extending ISystem<EntityStore>, it signals that any implementing class is a standard Entity-Component-System (ECS) system designed to operate on the server's central EntityStore.

This interface is a critical component of the physics engine's decoupled architecture. The engine's main update loop does not need to know about specific implementations like GravitySystem or FrictionSystem. Instead, it queries the system registry for all systems that implement IVelocityModifyingSystem. These systems are then executed in a deterministic order during the physics phase of the server tick.

This pattern allows developers to introduce new physical forces (e.g., wind, magical propulsion, knockback) by simply creating a new system that implements this interface, without modifying the core physics engine.

### Lifecycle & Ownership
As an interface, IVelocityModifyingSystem has no lifecycle. The following applies to all classes that **implement** this interface.

-   **Creation:** Instances are discovered and instantiated by the server's SystemManager during the bootstrap sequence, typically via reflection or explicit registration in a module definition.
-   **Scope:** An instance of an implementing system is a singleton within the server context. It persists for the entire duration of the server session.
-   **Destruction:** The instance is discarded and eligible for garbage collection during the server shutdown process.

## Internal State & Concurrency
-   **State:** Implementations of this interface should be **stateless**. They are transient processors that act upon the state stored in entity components (e.g., VelocityComponent, PhysicsStateComponent) within the EntityStore. Storing per-entity state within the system itself is a severe anti-pattern that breaks re-entrancy and thread safety.
-   **Thread Safety:** Implementations are not required to be internally thread-safe. The server's main loop guarantees that all IVelocityModifyingSystem instances are executed sequentially within a single, dedicated phase of the game tick. Concurrently modifying the EntityStore from these systems will lead to race conditions and unpredictable physics behavior.

## API Surface
The public contract is inherited entirely from the parent ISystem interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(EntityStore, float) | void | O(N) | Inherited from ISystem. Executes the system's logic on the provided EntityStore. The float argument represents the delta time for the current tick. |
| init() | void | O(1) | Inherited from ISystem. Called once by the SystemManager after instantiation to perform any necessary setup. |

## Integration Patterns

### Standard Usage
A developer creates a new system to apply a specific physical force, such as gravity. The class implements the marker interface, allowing the engine to automatically discover and integrate it into the physics pipeline.

```java
// Example: A system to apply basic gravity.
public class GravitySystem implements IVelocityModifyingSystem {

    private static final Vector3f GRAVITY_ACCELERATION = new Vector3f(0, -9.81f, 0);

    @Override
    public void process(EntityStore store, float deltaTime) {
        // Query for all entities that have both a Velocity and are affected by gravity.
        for (Entity entity : store.getEntitiesWith(VelocityComponent.class, GravityAffectedComponent.class)) {
            VelocityComponent velocity = entity.get(VelocityComponent.class);
            
            // Apply gravitational acceleration for the current tick.
            Vector3f currentVelocity = velocity.getLinear();
            currentVelocity.add(GRAVITY_ACCELERATION.mul(deltaTime, new Vector3f()));
            velocity.setLinear(currentVelocity);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Incorrect Categorization:** Do not implement this interface on systems that do not directly modify an entity's linear or angular velocity. For example, a system that updates collision volumes or resolves positional penetration should use a different interface. Mis-categorizing a system will cause it to execute in the wrong phase of the physics tick.
-   **Stateful Implementation:** Do not store entity-specific data as fields within the system class. This breaks the stateless processing model and will cause catastrophic bugs in a live server environment. All state must reside in components.

## Data Pipeline
IVelocityModifyingSystem implementations form a key stage in the server's physics simulation pipeline. They are executed after input processing but before the final position integration step.

> Flow:
> Server Tick Start -> Player Input Processing -> **All IVelocityModifyingSystem Instances (Gravity, Friction, etc.)** -> Collision Detection -> Penetration Resolution -> Position & Rotation Integration -> Server Tick End


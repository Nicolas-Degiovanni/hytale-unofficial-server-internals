---
description: Architectural reference for the ForceProvider interface, a core contract within the server-side physics engine.
---

# ForceProvider

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Contract / Strategy

## Definition
```java
// Signature
public interface ForceProvider {
```

## Architecture & Concepts
The ForceProvider interface defines a fundamental contract within the server-side physics simulation module. It embodies the Strategy pattern, abstracting the concept of a force that can be applied to a physical body. This design decouples the core physics integration loop from the specific sources of force, such as gravity, player input, explosions, or environmental effects like wind.

The physics engine maintains a collection of objects that implement this interface. During each simulation tick, for every active PhysicsBodyState, the engine iterates through its registered ForceProviders. Each provider is given the opportunity to inspect the body's state and apply forces via the provided ForceAccumulator. This allows for a highly extensible and composable system where new physical behaviors can be introduced by simply creating a new implementation of ForceProvider and registering it with the engine.

**WARNING:** Implementations of this interface are executed within the critical path of the physics simulation loop. Any performance degradation, blocking operations, or unhandled exceptions within a ForceProvider will directly impact server performance and simulation stability.

## Lifecycle & Ownership
As an interface, ForceProvider itself does not have a lifecycle. The lifecycle of a concrete implementation is determined entirely by its creator and the context in which it is used.

- **Creation:** ForceProvider implementations are instantiated by various systems depending on their purpose. A global force like gravity might be created as a singleton by the PhysicsModule during server startup. A temporary force, like one from an explosion, would be created on-demand by the combat or world event system.
- **Scope:** The scope varies dramatically. A GravityForceProvider is a long-lived, session-scoped object. An ExplosionForceProvider is a transient, short-lived object that is likely discarded after a few simulation ticks.
- **Destruction:** Implementations are subject to standard Java garbage collection. Long-lived providers are typically destroyed during module shutdown, while short-lived providers are destroyed once they are no longer referenced by the physics engine or the system that created them.

## Internal State & Concurrency
The interface itself is stateless. However, concrete implementations may be stateful.

- **State:** Implementations can range from completely stateless (e.g., a constant gravity provider) to highly stateful (e.g., a wind provider that uses a Perlin noise function based on world time). It is the responsibility of the implementer to manage this state correctly.
- **Thread Safety:** **NOT THREAD-SAFE.** The physics simulation is expected to run on a single, dedicated thread. All calls to ForceProvider implementations will be serialized by this main physics loop. Implementations **must not** assume they can be safely called from multiple threads. Any mutation of shared state from outside the physics thread must be handled with extreme care, typically using concurrent queues or other synchronization primitives to pass data to the physics thread. Direct mutation of a ForceProvider's state from another thread during a simulation tick is an undefined behavior and will lead to race conditions.

## API Surface
The public contract consists of a single method for applying forces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(PhysicsBodyState, ForceAccumulator, boolean) | void | O(N) | Called once per physics tick for a relevant body. The implementation calculates and applies forces to the ForceAccumulator. Complexity depends on the implementation. |

## Integration Patterns

### Standard Usage
The standard pattern is to create a class that implements the ForceProvider interface and register it with the appropriate physics world or system. The engine then automatically invokes the update method during the simulation.

```java
// Example of a simple gravity implementation
public class GravityForceProvider implements ForceProvider {
    private static final Vector3f GRAVITY = new Vector3f(0, -9.81f, 0);

    @Override
    public void update(PhysicsBodyState bodyState, ForceAccumulator accumulator, boolean isSleeping) {
        if (isSleeping || !bodyState.hasFiniteMass()) {
            return;
        }
        // Apply a constant downward force scaled by the body's mass
        Vector3f gravityForce = new Vector3f(GRAVITY);
        gravityForce.mul(bodyState.getMass());
        accumulator.addForce(gravityForce);
    }
}

// Registration (conceptual)
PhysicsSystem physics = context.getService(PhysicsSystem.class);
physics.addGlobalForceProvider(new GravityForceProvider());
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never perform file I/O, network requests, or any other long-running or blocking operation within the update method. This will stall the entire server physics thread.
- **Stateful Instability:** Avoid creating implementations with internal state that is not reset or managed correctly between calls. The update method may be called for thousands of different bodies in a single tick, and leftover state from one call can incorrectly affect the next.
- **Direct State Modification:** Do not directly modify the PhysicsBodyState. All physical changes must be applied through the ForceAccumulator. The integrator uses the accumulated forces to resolve the final state at the end of the tick. Bypassing the accumulator breaks the simulation.

## Data Pipeline
The ForceProvider is a processing stage within the larger physics simulation pipeline.

> Flow:
> Physics Tick Start -> Iterate Active Bodies -> For each body, invoke all registered **ForceProvider.update()** methods -> Forces are added to a ForceAccumulator -> Integrator uses accumulated forces to calculate new velocity and position -> PhysicsBodyState is updated -> Physics Tick End


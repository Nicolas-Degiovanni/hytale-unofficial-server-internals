---
description: Architectural reference for SpawnEffect
---

# SpawnEffect

**Package:** com.hypixel.hytale.server.npc.role
**Type:** Behavioral Interface

## Definition
```java
// Signature
public interface SpawnEffect {
```

## Architecture & Concepts
The SpawnEffect interface defines a behavioral contract for objects that need to trigger a visual particle effect upon their creation or appearance in the world. It follows a pattern similar to a "mixin" or "trait", providing a default implementation for the core logic while allowing implementers to define the specific parameters of the effect.

This interface acts as a standardized bridge between an entity's lifecycle events (e.g., spawning) and the server's core visual effect systems. Its primary responsibility is to:
1.  Define the visual properties of a spawn effect (particle type, offset, and view distance).
2.  Provide a concrete implementation for finding nearby players and broadcasting the particle effect to them.

By implementing this interface, a class, typically an NPC role or a specific entity type, gains the capability to display a spawn effect without containing the complex logic for spatial queries or network replication. It effectively decouples the *what* (the particle effect's parameters) from the *how* (the engine-level implementation of spawning it).

## Lifecycle & Ownership
As an interface, SpawnEffect does not have its own lifecycle. Its behavior and lifetime are entirely governed by the object that implements it.

-   **Creation:** This interface is not instantiated. A concrete class that `implements SpawnEffect` is created by its owner, for example, an NPC factory or a game logic controller.
-   **Scope:** The behavior defined by SpawnEffect is active for the entire lifetime of the implementing object.
-   **Destruction:** No special destruction or cleanup is required. The behavior ceases to exist when the implementing object is garbage collected.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. The `spawnEffect` default method is a pure function with respect to the interface, operating exclusively on its method parameters and the data provided by the implementing class via the abstract getter methods. All state (particle name, offset, distance) is owned by the implementing class.

-   **Thread Safety:** **CRITICAL WARNING:** This interface is **not thread-safe** and is designed to be used exclusively from the main server world-tick thread. The `spawnEffect` method internally uses `SpatialResource.getThreadLocalReferenceList()`, which relies on a thread-local object pool for performance. Calling this method from any other thread will result in undefined behavior, likely a `NullPointerException` or a resource leak. All invocations must be synchronized with the main game loop.

## API Surface
The public contract consists of abstract methods for configuration and a default method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnParticles() | String | O(1) | **Contract:** Must return the asset name of the particle effect. |
| getSpawnParticleOffset() | Vector3d | O(1) | **Contract:** Must return the positional offset relative to the spawn location. |
| getSpawnViewDistance() | double | O(1) | **Contract:** Must return the radius in which players will see the effect. |
| spawnEffect(position, rotation, accessor) | void | O(N) | Triggers the particle effect logic. N is the number of players within the view distance. Throws exceptions if parameters are null. |

## Integration Patterns

### Standard Usage
The intended pattern is for a class, such as an NPC role definition, to implement the interface and provide concrete values for the particle effect. The `spawnEffect` method is then called during the entity's own spawning logic.

```java
// In an NPC's spawn-handling code
// Assume 'myNpcRole' is an object that implements SpawnEffect

if (myNpcRole instanceof SpawnEffect) {
    Vector3d spawnPosition = npc.getPosition();
    Vector3f spawnRotation = npc.getRotation();
    ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();

    // The cast is conventional to signal intent
    ((SpawnEffect) myNpcRole).spawnEffect(spawnPosition, spawnRotation, accessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Asynchronous Invocation:** Never call `spawnEffect` from a separate thread, a future, or any asynchronous task. The reliance on thread-local storage makes this extremely hazardous.
-   **Stateful Implementations:** The getter methods (`getSpawnParticles`, etc.) should be simple accessors. Avoid performing heavy computations or I/O within them, as they are called on the critical server tick path.
-   **External Logic Duplication:** Do not re-implement the logic of `spawnEffect` outside of this interface. The default method is optimized and guaranteed to be compatible with the engine's spatial partitioning and particle systems.

## Data Pipeline
The `spawnEffect` method orchestrates a flow of data and commands across several core server modules.

> Flow:
> Entity Spawn Event -> `spawnEffect(position, ...)` -> **SpawnEffect** calculates final particle position -> `SpatialResource` is queried for nearby players -> `ParticleUtil` is invoked with particle name and target players -> Particle System serializes effect data -> Network packets are sent to relevant clients.


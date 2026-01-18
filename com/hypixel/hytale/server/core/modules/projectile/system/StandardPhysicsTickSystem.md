---
description: Architectural reference for StandardPhysicsTickSystem
---

# StandardPhysicsTickSystem

**Package:** com.hypixel.hytale.server.core.modules.projectile.system
**Type:** System Component

## Definition
```java
// Signature
public class StandardPhysicsTickSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts

The StandardPhysicsTickSystem is a core processor within the server-side Entity Component System (ECS) framework, responsible for simulating the physical behavior of entities. It specifically targets entities equipped with a StandardPhysicsProvider component, which typically includes projectiles and other simple, non-character dynamic objects.

This system operates once per server tick for every entity matching its component query. Its primary function is to advance an entity's state by applying forces, resolving collisions, and updating its position and velocity. It integrates several physical concepts:

*   **Force Application:** Calculates the effects of gravity, drag, and other external forces.
*   **Collision Detection:** Performs iterative collision checks against both the static world geometry (blocks) and other dynamic entities.
*   **Collision Response:** Implements behaviors such as bouncing, sliding, friction, and resting upon impact.
*   **Fluid Dynamics:** Manages basic interactions when an entity enters or leaves a fluid volume.

The system is a critical link between an entity's raw data components (like TransformComponent and Velocity) and its emergent behavior in the game world. It reads the entity's current state, consults the world environment, computes the next state, and writes the results back to the entity's components.

### Lifecycle & Ownership

-   **Creation:** Instantiated automatically by the ECS SystemGraph during the initialization of a server World. It is not intended for manual creation by developers.
-   **Scope:** The lifecycle of a StandardPhysicsTickSystem instance is bound to its parent World. It persists for the entire duration of the world simulation.
-   **Destruction:** The instance is marked for garbage collection when the corresponding World is unloaded or the server shuts down.

## Internal State & Concurrency

-   **State:** This class is fundamentally stateless. It does not retain any data between invocations of its tick method. All state is read from and written to the ECS components of the entity being processed or from shared, read-only resources like TimeResource. The internal query and dependencies fields are immutable after construction.

-   **Thread Safety:** This system is not thread-safe and is designed to be operated by a single-threaded ECS scheduler. The Hytale ECS framework guarantees that the tick method is executed in a deterministic order and without concurrent access, ensuring that all component reads and writes are safe within the scope of a single world update cycle.

## API Surface

The public contract is primarily defined by its base class, EntityTickingSystem. Direct interaction with its methods is reserved for the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Set | O(1) | Returns the set of dependencies that dictate its execution order relative to other systems. |
| getQuery() | Query | O(1) | Returns the ECS query used to identify which entities this system should process. |
| tick(dt, index, chunk, store, commands) | void | O(N) | Executes one simulation step for a single entity. Complexity is dominated by collision detection algorithms. |

## Integration Patterns

### Standard Usage

Developers do not interact with this class directly. To make an entity subject to this physics simulation, you must attach the required components to it during its creation. The system will automatically discover and process the entity on each server tick.

```java
// Example: Making an entity physical
// This code would exist within an entity factory or spawner.

// The CommandBuffer is used to schedule entity modifications.
CommandBuffer<EntityStore> commands = ...;
Ref<EntityStore> entityRef = commands.createEntity();

// Attach the necessary components for the system's query to match.
commands.addComponent(entityRef, new StandardPhysicsProvider(...));
commands.addComponent(entityRef, new TransformComponent(...));
commands.addComponent(entityRef, new Velocity(...));
commands.addComponent(entityRef, new HeadRotation());
commands.addComponent(entityRef, new BoundingBox(...));

// The StandardPhysicsTickSystem will now process this entity automatically.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new StandardPhysicsTickSystem()`. The ECS framework is solely responsible for its lifecycle management. Doing so will result in a non-functional system that is not registered with the game loop.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the ECS scheduler, breaks dependency ordering, and will lead to severe concurrency issues, data corruption, and unpredictable behavior.
-   **Incomplete Component Set:** An entity will be completely ignored by this system if it lacks any of the components defined in the internal query (StandardPhysicsProvider, TransformComponent, Velocity, etc.). This is a common source of bugs where an entity fails to move.

## Data Pipeline

This system acts as a transformation node within the main server game loop. It processes entity state data rather than handling I/O.

> Flow:
> **Input Components** (Transform, Velocity, StandardPhysicsProvider) -> **StandardPhysicsTickSystem** (Force Calculation -> Collision Casting -> State Resolution) -> **Output Components** (Updated Transform, Updated Velocity) -> **CommandBuffer** (Impact/Bounce Events)


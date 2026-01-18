---
description: Architectural reference for Velocity
---

# Velocity

**Package:** com.hypixel.hytale.server.core.modules.physics.component
**Type:** Data Component

## Definition
```java
// Signature
public class Velocity implements Component<EntityStore> {
```

## Architecture & Concepts
The Velocity component is a fundamental data container within the server-side Entity Component System (ECS). It is not a system or a service; it is pure state attached to an entity, representing its movement through the world.

Its primary architectural role is to decouple the *intent* of movement from its *execution*. Systems like combat, AI, or player input do not directly manipulate an entity's position. Instead, they modify the entity's Velocity component. A dedicated Physics System then consumes this state during the world tick to calculate the final position update.

This component makes a critical distinction between server-authoritative velocity and client-reported velocity:
- **velocity:** The canonical, server-side velocity vector. This is the ground truth for all physics calculations and game logic.
- **clientVelocity:** A separate vector storing the velocity as reported by the game client. This is used primarily for server-side validation, anti-cheat detection, and network reconciliation logic.
- **instructions:** A command queue for deferred velocity changes. This pattern allows multiple systems to submit velocity modifications within a single tick. The Physics System can then process these instructions in a deterministic order, resolving potentially conflicting forces (e.g., a knockback effect and player input) according to game rules.

The static CODEC field signifies that this component is serializable, enabling its state to be persisted to disk during world saves or replicated over the network.

## Lifecycle & Ownership
- **Creation:** A Velocity component is instantiated and attached to an entity by the ECS framework, typically via the EntityModule. This occurs either during the initial creation of a physics-enabled entity or when an entity gains physics properties dynamically.
- **Scope:** The lifecycle of a Velocity component is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed and removed from the EntityStore. There is no explicit destruction method to call; its memory is managed by the ECS and the JVM.

## Internal State & Concurrency
- **State:** The Velocity component is highly mutable. Its core state, including the velocity vectors and the list of instructions, is designed to be updated frequently, often multiple times per game tick.
- **Thread Safety:** This class is **not thread-safe**. All internal fields are accessed without any synchronization mechanisms. It is architected to be owned and modified exclusively by the single thread responsible for ticking the entity's world (e.g., the main server thread).

**WARNING:** Concurrent modification from multiple threads will lead to race conditions, physics instability, and world corruption. All interactions with a Velocity component must be synchronized with the main game loop.

## API Surface
The public API is designed for direct state manipulation by trusted server systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(double x, double y, double z) | void | O(1) | Overwrites the server-authoritative velocity vector. |
| addForce(Vector3d force) | void | O(1) | Applies an instantaneous change to the server-authoritative velocity. |
| setClient(double x, double y, double z) | void | O(1) | Overwrites the client-reported velocity. Used by network handlers. |
| addInstruction(velocity, config, type) | void | O(1) | Enqueues a deferred velocity change to be processed by the Physics System. |
| getInstructions() | List | O(1) | Returns a direct reference to the internal list of instructions. |
| getVelocity() | Vector3d | O(1) | Returns a direct reference to the server-authoritative velocity vector. |

**WARNING:** The methods getInstructions and getVelocity return mutable, internal state. Modifying these returned objects directly can bypass intended logic and is strongly discouraged. Use the provided API methods like addForce or addInstruction instead.

## Integration Patterns

### Standard Usage
Systems should retrieve the component from an entity and use the API to apply forces or queue instructions.

```java
// Example: Applying a knockback force within a combat system
Entity target = ...;
Velocity velocityComponent = target.getComponent(Velocity.getComponentType());

if (velocityComponent != null) {
    Vector3d knockbackForce = new Vector3d(0, 5.0, 10.0);
    velocityComponent.addForce(knockbackForce);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a component with `new Velocity()`. Components are managed by the ECS. Use `entity.addComponent(...)` or equivalent entity management APIs.
- **Cross-Thread Modification:** Do not access or modify a Velocity component from an asynchronous task or a different thread without explicit synchronization with the main world tick.
- **State Confusion:** Do not write game logic that reads from `clientVelocity`. This field is exclusively for network reconciliation. All gameplay simulation must use the main `velocity` field.
- **Instruction List Mismanagement:** Do not manually clear the instruction list. The Physics System is the owner of this process and will clear the list after processing the instructions each tick.

## Data Pipeline
The Velocity component acts as a stateful buffer between game logic and the physics simulation.

> **Server-Side Force Application:**
> Game Logic (e.g., Skill System) -> `velocity.addForce()` -> **Velocity Component State** -> Physics System (reads state) -> Entity Position Update

> **Deferred Instruction Processing:**
> AI System -> `velocity.addInstruction(...)` -> **Velocity Component Instruction Queue** -> Physics System (dequeues and applies instructions) -> **Velocity Component State** -> Entity Position Update

> **Client Input Reconciliation:**
> Network Packet -> Packet Handler -> `velocity.setClient(...)` -> **Velocity Component State** -> Anti-Cheat/Reconciliation System (compares `velocity` and `clientVelocity`) -> Corrective Action (if necessary)


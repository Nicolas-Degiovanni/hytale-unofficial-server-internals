---
description: Architectural reference for PredictedProjectile
---

# PredictedProjectile

**Package:** com.hypixel.hytale.server.core.modules.projectile.component
**Type:** Component (Data-Only)

## Definition
```java
// Signature
public class PredictedProjectile implements Component<EntityStore> {
```

## Architecture & Concepts
The PredictedProjectile component is a critical part of the server's client-side prediction system for projectiles. It does not contain any logic itself; instead, it acts as a data-holding **marker component** within Hytale's Entity-Component-System (ECS) architecture.

Its primary function is to flag a server-side projectile entity as having a client-side counterpart that was spawned predictively. The component holds a universally unique identifier (UUID) that serves as a synchronization key between the server-authoritative entity and the client's speculative visual effect.

When a player fires a weapon, the client can immediately render the projectile without waiting for the server's response, reducing perceived latency. The server, upon processing the fire command, creates the actual projectile entity, attaches this component with a new UUID, and sends that UUID back to the client. The client then uses this UUID to link its predicted projectile to the authoritative one, allowing for future corrections or hit confirmations from the server.

This component exists exclusively on the server.

### Lifecycle & Ownership
- **Creation:** Instantiated by the ProjectileModule or a related combat system on the server immediately after a player action generates a projectile that supports prediction. It is attached to the new projectile entity as part of its initial setup.
- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the projectile exists in the world.
- **Destruction:** The component is destroyed automatically and implicitly when its parent entity is removed from the EntityStore. This occurs when the projectile hits a target, expires, or is otherwise cleaned up by the server.

## Internal State & Concurrency
- **State:** **Immutable**. The internal UUID is a final field, set only once during construction. The object's state cannot be modified after creation, which makes it a simple and reliable data carrier.
- **Thread Safety:** **Inherently Thread-Safe**. As an immutable object, an instance of PredictedProjectile can be safely passed between and read by multiple threads without locks or other synchronization primitives. Note that any modifications to the parent entity must still adhere to the concurrency rules of the governing EntityStore.

## API Surface
The public API is minimal, reflecting its role as a simple data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type definition for this component from the ProjectileModule. |
| getUuid() | UUID | O(1) | Returns the unique identifier used to correlate this server entity with its client-side prediction. |
| clone() | Component | O(1) | Returns the same instance. This is a valid optimization for immutable objects. |

## Integration Patterns

### Standard Usage
This component is created and attached to an entity by a server-side system responsible for spawning projectiles. The UUID is then typically serialized into a network packet sent to the originating client.

```java
// Example from a server-side combat or ability system
UUID predictionKey = UUID.randomUUID();
Entity projectile = world.createEntity();

// Attach this component to mark the entity for prediction synchronization
projectile.addComponent(new PredictedProjectile(predictionKey));

// Attach other necessary components (e.g., Position, Velocity, Damage)
// ...

// Inform the client about the authoritative entity and provide the key
PacketSpawnPredictedProjectile packet = new PacketSpawnPredictedProjectile(projectile.getId(), predictionKey, ...);
networkService.send(player, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Modification:** Do not use reflection to change the UUID after construction. The stability of the prediction system relies on this key being immutable.
- **Component Re-use:** Do not detach this component from a destroyed entity and attach it to a new one. Each predicted projectile must have its own unique instance with a new UUID to prevent synchronization conflicts.
- **Client-Side Instantiation:** This is a server-only component. Clients receive the UUID as raw data within a network packet; they do not instantiate or manage this specific component class.

## Data Pipeline
PredictedProjectile does not process data; it is a payload that enables a data flow between the server and client to reduce latency.

> Flow:
> Player Input (Client) -> Fire Weapon Command (Network) -> **ProjectileModule (Server)** -> Creates Entity + **PredictedProjectile** -> Network Packet with UUID -> Client Prediction System -> Correlates and corrects the visual projectile.


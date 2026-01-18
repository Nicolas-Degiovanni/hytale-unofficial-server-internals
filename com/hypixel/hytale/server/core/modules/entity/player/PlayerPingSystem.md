---
description: Architectural reference for PlayerPingSystem
---

# PlayerPingSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System Component

## Definition
```java
// Signature
public class PlayerPingSystem extends EntityTickingSystem<EntityStore> implements RunWhenPausedSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerPingSystem is a server-side system within the Hytale Entity Component System (ECS) framework. Its sole responsibility is to drive the network ping lifecycle for all connected players. It achieves this by iterating over every entity that possesses a **PlayerRef** component.

This system is a critical component for maintaining stable client-server connections. By implementing the **RunWhenPausedSystem** interface, it ensures that player connection health is monitored even when the main game simulation is paused. This prevents players from incorrectly timing out during server maintenance or debug pauses.

Architecturally, this system is designed to run at the end of the server tick. Its inclusion in the **SEND_PACKET_GROUP** and its **RootDependency.lastSet** dependency configuration explicitly places it after all other game logic has been processed. This guarantees that any network packets it generates reflect the final, authoritative state of the world for that tick.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS System Scheduler during the server bootstrap sequence. This system is not intended for manual creation.
- **Scope:** Session-scoped. It persists for the entire lifetime of the server process. Its lifecycle is directly managed by the parent ECS World.
- **Destruction:** Decommissioned and garbage collected only when the server shuts down and the primary ECS World is destroyed.

## Internal State & Concurrency
- **State:** The PlayerPingSystem is entirely stateless. It contains no mutable instance fields and does not cache any data between invocations. All operations are performed on data owned by the components of the entities it processes.
- **Thread Safety:** This system is inherently thread-safe. The `tick` method is a pure function with respect to the system's own state. The ECS scheduler may execute this system's logic in parallel across different chunks of entities, as governed by the `isParallel` method. Concurrency control for the underlying player data is the responsibility of the **PacketHandler** object accessed via the PlayerRef component.

## API Surface
The public contract of this class is intended for consumption by the ECS framework, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(1) per entity | Framework-invoked method. For a single entity, it retrieves the PlayerRef component and delegates the ping logic to its associated PacketHandler. |
| getQuery() | Query | O(1) | Defines the set of entities this system operates on: all entities with a PlayerRef component. |
| getDependencies() | Set | O(1) | Declares that this system must run after all other systems in the main dependency graph. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. It is automatically registered with and executed by the server's core scheduler. To make an entity eligible for processing by this system, simply attach a **PlayerRef** component to it. The system will then discover and manage it automatically.

```java
// A developer does not write this code.
// This is a conceptual representation of how the ECS scheduler invokes the system.

// For each ArchetypeChunk containing entities with a PlayerRef component:
for (int i = 0; i < chunk.size(); i++) {
    playerPingSystem.tick(dt, i, chunk, store, commandBuffer);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerPingSystem()`. The ECS framework is solely responsible for its lifecycle. Direct instantiation will result in a non-functional object that is not registered with the scheduler.
- **Manual Invocation:** Do not call the `tick` method from application code. Doing so bypasses the scheduler's dependency resolution and thread management, which can lead to severe race conditions and unpredictable network behavior.
- **Altering Dependencies:** Modifying the system to remove its `RootDependency.lastSet` dependency is dangerous. This could cause the ping mechanism to operate on an inconsistent or intermediate game state, leading to desynchronization or other network errors.

## Data Pipeline
The PlayerPingSystem acts as a trigger within the server's data flow, initiating the ping check process which culminates in network traffic.

> Flow:
> Server Game Loop -> ECS Scheduler -> **PlayerPingSystem.tick()** -> PlayerRef.getPacketHandler().tickPing() -> Network Layer -> Outbound UDP Packet to Client


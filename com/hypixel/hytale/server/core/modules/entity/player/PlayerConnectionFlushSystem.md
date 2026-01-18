---
description: Architectural reference for PlayerConnectionFlushSystem
---

# PlayerConnectionFlushSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System Component

## Definition
```java
// Signature
public class PlayerConnectionFlushSystem extends EntityTickingSystem<EntityStore> implements RunWhenPausedSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerConnectionFlushSystem is a critical component within the server's Entity Component System (ECS) framework. It acts as the final gatekeeper in the per-tick network data pipeline, ensuring that all buffered packets for a player are sent.

Architecturally, this is a "commit" or "flush" system. Its primary responsibility is not to generate data, but to trigger the dispatch of data buffered by other systems throughout the game tick. Its position in the execution order is paramount; it is explicitly configured to run *after* all other systems that might generate network packets. This prevents partial or incomplete world state from being sent to the client.

The implementation of RunWhenPausedSystem is a key design choice. It guarantees that network communication remains active even if the core game simulation is paused. This allows players to remain connected, receive keep-alive pings, and interact with non-gameplay systems (like chat) while the world is frozen.

The system queries for all entities possessing a PlayerRef component, ensuring it operates exclusively on player entities.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS SystemGraph builder during server initialization. The framework discovers this system and integrates it into the execution schedule based on its declared dependencies.
- **Scope:** Session-scoped. A single instance of this system persists for the entire lifetime of the server. It is stateless and reused every tick.
- **Destruction:** The instance is discarded and eligible for garbage collection only when the server's main ECS world is shut down.

## Internal State & Concurrency
- **State:** This system is **stateless**. Its only instance variable, componentType, is an immutable reference injected during construction. It does not cache or store any data between ticks or across entities. All operations are performed on state owned by the components it queries.
- **Thread Safety:** Conditionally thread-safe. The isParallel method delegates the decision to a utility function, indicating it can be scheduled to run concurrently across different blocks of player entities. The safety of this parallel execution is predicated on the thread safety of the PacketHandler's tryFlush method, which is retrieved from the PlayerRef component. The system itself introduces no race conditions as it holds no mutable state.

## API Surface
The public API is designed for consumption by the ECS scheduler, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Set | O(1) | Returns the declarative rules for scheduling, placing this system at the end of the network pipeline. |
| getQuery() | Query | O(1) | Returns the query that selects all entities with a PlayerRef component. |
| tick(...) | void | O(N) | The main execution logic. Iterates over all matched player entities and invokes tryFlush on their connection. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Its functionality is implicitly invoked by the server's main game loop scheduler. To have packets sent by this system, other systems should add packets to a player's PacketHandler buffer.

```java
// In another system that runs BEFORE PlayerConnectionFlushSystem:

// 1. Get the player's PacketHandler
PacketHandler handler = playerRef.getPacketHandler();

// 2. Queue a packet for sending. It will be buffered.
handler.send(somePacket);

// 3. Later in the tick, PlayerConnectionFlushSystem will automatically run
//    and call handler.tryFlush(), sending the packet.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerConnectionFlushSystem()`. The ECS framework manages its lifecycle. Direct instantiation will result in a non-functional object that is not registered with the game loop.
- **Manual Invocation:** Calling the tick method manually will bypass the dependency scheduler. This can cause network packets to be sent out-of-order or before the game state for the current tick has been fully computed, leading to severe desynchronization bugs.
- **Incorrect Dependency Ordering:** Creating a new system that sends packets but scheduling it to run *after* PlayerConnectionFlushSystem will cause its packets to be delayed by an entire tick.

## Data Pipeline
This system is a terminal trigger in the server's outgoing data flow. It does not transform data but ensures its final transmission.

> Flow:
> Game Logic Systems -> Generate Network Packet -> PlayerRef.getPacketHandler().send(packet) -> Internal Network Buffer -> **PlayerConnectionFlushSystem** -> PacketHandler.tryFlush() -> OS Socket Buffer -> Client


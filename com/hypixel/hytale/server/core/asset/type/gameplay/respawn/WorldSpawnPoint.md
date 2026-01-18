---
description: Architectural reference for WorldSpawnPoint
---

# WorldSpawnPoint

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay.respawn
**Type:** Singleton

## Definition
```java
// Signature
public class WorldSpawnPoint implements RespawnController {
```

## Architecture & Concepts
The WorldSpawnPoint class is a concrete implementation of the **Strategy Pattern**, conforming to the RespawnController interface. Its singular purpose is to define a respawn strategy that places a player at the world's designated default spawn point.

Architecturally, this class acts as a stateless service node within the server's core gameplay loop. It does not perform actions directly. Instead, it integrates with the Entity Component System (ECS) by issuing commands through a CommandBuffer. When `respawnPlayer` is invoked, it does not immediately change the player's position. It enqueues a `Teleport` component to be added to the player entity. This command is processed by the relevant ECS system at a safe point later in the game tick, ensuring transactional integrity and preventing mid-tick state corruption.

The class is designed as a singleton, exposed via the static `INSTANCE` field. This reinforces its nature as a pure, stateless function container, ensuring that all server threads use the exact same logic without the overhead of repeated instantiation. The associated `CODEC` allows this strategy to be referenced and deserialized from game data files, making it a data-driven component of the gameplay ruleset.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the JVM during class loading via the `public static final INSTANCE` field. This occurs very early in the server's startup sequence.
- **Scope:** Application-wide. A single instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is destroyed only when the JVM shuts down. There is no manual cleanup logic.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to its methods.
- **Thread Safety:** Inherently **thread-safe**. As a stateless object, it can be safely accessed and invoked from any thread without locks or synchronization. The responsibility for thread safety lies with the mutable objects passed into it, such as the World and CommandBuffer, which are managed by the calling game tick scheduler.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| respawnPlayer(World, Ref, CommandBuffer) | void | O(1) | Determines the world's spawn point and enqueues a command to teleport the specified player entity. This is a non-blocking, deferred operation. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most game logic developers. It is designed to be configured within game assets and is called by the server's core systems that manage player lifecycle events, such as post-death processing.

```java
// This logic resides within a higher-level system managing player death.
// The 'controller' is typically loaded from a data asset.
RespawnController controller = WorldSpawnPoint.INSTANCE;

// The system provides the current world, a reference to the player entity,
// and the command buffer for the current tick.
controller.respawnPlayer(currentWorld, playerEntityRef, tickCommandBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new WorldSpawnPoint()`. The class is a singleton and must only be accessed via `WorldSpawnPoint.INSTANCE` to guarantee a single, shared instance.
- **Stateful Modification:** Do not modify this class to hold any per-player or per-world state. Its stateless design is critical for its function as a reusable, data-driven strategy.

## Data Pipeline
WorldSpawnPoint acts as a command generator in a larger process flow. It does not transform data but rather initiates a state change command based on its inputs.

> Flow:
> Player Death Event → Core Respawn System → **WorldSpawnPoint.respawnPlayer()** → CommandBuffer Enqueues Teleport Component → ECS Command Processor → Player Entity Position Updated


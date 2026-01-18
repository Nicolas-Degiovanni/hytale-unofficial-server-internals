---
description: Architectural reference for UpdateSleepPacketSystem
---

# UpdateSleepPacketSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.player
**Type:** System Component

## Definition
```java
// Signature
public class UpdateSleepPacketSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts

The UpdateSleepPacketSystem is a server-side ECS (Entity Component System) system responsible for synchronizing a player's sleep state with their game client. It acts as a translator, converting abstract server-side sleep-related component data into concrete network packets that the client UI can render.

This system operates on a fixed schedule, querying for all entities that represent players currently involved in the sleep process. Its primary function is to construct and dispatch the UpdateSleepState packet, which dictates the visibility and state of the sleep interface, including fade-to-black effects and multiplayer status information.

By isolating the packet creation logic, this system decouples the core gameplay mechanics of sleeping (managed by other systems) from the network-level client representation. It is a critical component in the "View" layer of the server's sleep feature, ensuring the player's visual experience accurately reflects the underlying simulation state.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central ECS System Scheduler during the world initialization or module loading phase. It is not created on a per-player or per-request basis.
- **Scope:** Application-scoped. A single instance of this system persists for the entire lifetime of the server world.
- **Destruction:** The instance is destroyed and garbage collected only upon server shutdown or when the world is unloaded.

## Internal State & Concurrency
- **State:** This system is fundamentally **stateless**. It holds no mutable instance fields and does not cache data between ticks. All information required for its operation is read directly from ECS components (like PlayerSomnolence) and resources (like WorldSomnolence) during each execution of the tick method.
- **Thread Safety:** **Not thread-safe.** This system is designed to be executed exclusively by the main server thread or a designated ECS worker thread. Direct invocation from other threads will lead to race conditions and data corruption, as it performs non-atomic reads on shared component stores. The ECS scheduler guarantees safe execution context.

## API Surface

The public API is intended for consumption by the ECS framework, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will process. |
| tick(...) | void | O(N) | Framework-invoked method. Executes the core logic for a single entity matching the query. N is the number of players in the world during multiplayer status checks. |

## Integration Patterns

### Standard Usage

Direct interaction with this system is not a standard development practice. Integration is achieved declaratively by attaching the required components to a player entity. The system will automatically discover and process the entity.

```java
// A developer enables this system's functionality for a player
// by ensuring the entity has the correct components.
// The system itself is never directly referenced.

// Pseudo-code for adding a player to the sleep system
Entity playerEntity = ...;
CommandBuffer commands = ...;

commands.addComponent(playerEntity, new PlayerSomnolence());
commands.addComponent(playerEntity, new SleepTracker());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UpdateSleepPacketSystem()`. The ECS framework is solely responsible for the lifecycle of systems. Manual creation will result in a non-functional object that is not registered with the game loop.
- **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the ECS scheduler, breaks execution order guarantees, and introduces severe thread-safety risks.
- **State Modification:** Do not attempt to modify the system to store state in instance fields. This violates the stateless nature of ECS systems and will cause unpredictable behavior in a live server environment.

## Data Pipeline

This system's primary role is to process data from the ECS and transform it into network packets.

> Flow:
> ECS State (PlayerSomnolence, WorldSomnolence) -> **UpdateSleepPacketSystem** -> UpdateSleepState Packet Construction -> PlayerRef Network Handler -> Client UI Update


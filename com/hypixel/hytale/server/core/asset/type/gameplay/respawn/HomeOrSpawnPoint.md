---
description: Architectural reference for HomeOrSpawnPoint
---

# HomeOrSpawnPoint

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay.respawn
**Type:** Singleton

## Definition
```java
// Signature
public class HomeOrSpawnPoint implements RespawnController {
```

## Architecture & Concepts
The HomeOrSpawnPoint class is a concrete implementation of the RespawnController strategy interface. Its sole responsibility is to define a specific respawn behavior: relocating a player entity to their designated "home" position or, if one is not set, to the world's default spawn point.

Architecturally, this class acts as a stateless, logic-only service within the server's core gameplay loop. It is invoked by higher-level systems that manage player death and respawn events. By implementing the RespawnController interface, it allows the server's behavior to be configured declaratively. Different worlds or game modes can swap out this implementation for another without altering the core death-handling code.

Its primary mechanism of action is not to directly manipulate the player's state, but to issue a **Teleport** command via a CommandBuffer. This adheres to the engine's Entity-Component-System (ECS) design, ensuring that state changes are deferred and processed systematically by dedicated systems at the end of the game tick.

## Lifecycle & Ownership
- **Creation:** Instantiated statically at class-loading time. The public static final INSTANCE field ensures a single, globally accessible object is created by the JVM.
- **Scope:** Application-level singleton. The INSTANCE persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected by the JVM during the final stages of server shutdown. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is purely determined by the arguments passed to its methods. All necessary context, such as the world and player reference, is provided at invocation time.
- **Thread Safety:** Inherently thread-safe. As a stateless singleton, it can be safely invoked from any thread without risk of race conditions or data corruption. No internal locking mechanisms are necessary or used.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| respawnPlayer(World, Ref, CommandBuffer) | void | O(1) | Executes the respawn logic. Determines the target location by querying the Player component and issues a Teleport command to the ECS. Complexity is constant time relative to this class, but depends on the performance of underlying systems it calls. |

## Integration Patterns

### Standard Usage
This controller is not intended for direct, frequent invocation by game logic developers. It is designed to be configured within a gameplay asset (e.g., a world or ruleset file) and is invoked automatically by the server's core death processing system.

```java
// In a system that handles player death events:
// The specific controller is retrieved from game configuration.
RespawnController controller = HomeOrSpawnPoint.INSTANCE;

// The system invokes the controller, passing the necessary context.
controller.respawnPlayer(currentWorld, playerEntityRef, ecsCommandBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HomeOrSpawnPoint()`. This class is a singleton. Always use the static `HomeOrSpawnPoint.INSTANCE` field to ensure a single, shared instance is used throughout the server.
- **State Assumption:** Do not build logic that assumes this class holds or remembers any state between calls. It is a pure function host; its output depends only on its inputs for a given call.

## Data Pipeline
This component acts as a command generator within a larger data and control flow. It translates a high-level gameplay event (a player needs to respawn) into a low-level ECS command.

> Flow:
> Player Death Event -> Core Respawn System -> **HomeOrSpawnPoint.respawnPlayer()** -> CommandBuffer.addComponent(Teleport) -> ECS Tick Processor -> TeleportSystem -> Player Position Update


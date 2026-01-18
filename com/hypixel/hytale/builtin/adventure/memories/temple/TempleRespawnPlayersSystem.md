---
description: Architectural reference for TempleRespawnPlayersSystem
---

# TempleRespawnPlayersSystem

**Package:** com.hypixel.hytale.builtin.adventure.memories.temple
**Type:** System Component

## Definition
```java
// Signature
public class TempleRespawnPlayersSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts
The TempleRespawnPlayersSystem is a server-side Entity Component System (ECS) responsible for handling player respawns within a specific gameplay zone, identified as the "Forgotten Temple". It functions as a safety mechanism, preventing players from falling out of the world or becoming stuck in unrecoverable locations below the playable area.

This system operates on a delayed tick, inheriting from DelayedEntitySystem. The constructor argument of 1.0F specifies that its logic executes once per second, not on every game tick. This is an optimization, as the check for a player's vertical position does not require high-frequency updates.

Its core responsibility is to query for all entities that are players (possessing a PlayerRef and TransformComponent), check their Y-coordinate, and if it falls below a configurable threshold, issue a command to teleport them back to a valid spawn point. The system is tightly coupled to the world's configuration, specifically the ForgottenTempleConfig, from which it derives the minimum Y-level and the sound to play upon respawn.

## Lifecycle & Ownership
-   **Creation:** This system is instantiated by the server's core ECS framework during world initialization. It is discovered and registered automatically as part of the builtin adventure gameplay module. Developers do not create instances of this class manually.
-   **Scope:** An instance of TempleRespawnPlayersSystem persists for the entire lifetime of a game world. Its lifecycle is directly managed by the world's System Runner.
-   **Destruction:** The instance is marked for garbage collection when the server world is unloaded or during a full server shutdown.

## Internal State & Concurrency
-   **State:** This class is entirely stateless. It contains no mutable instance fields and relies solely on the data passed into its `tick` method for execution. All configuration and entity data is external, which is a best practice for ECS systems.
-   **Thread Safety:** The system is designed to be run by a single-threaded or job-based ECS scheduler. It is not thread-safe for arbitrary concurrent calls. All state mutations are deferred by writing components to the CommandBuffer. This pattern ensures that all changes are applied at a safe synchronization point in the game loop, preventing race conditions.

## API Surface
The public API is minimal and intended for framework-level interaction only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmdBuffer) | void | O(1) | Executes the respawn logic for a single entity. Called by the ECS runner. |
| getQuery() | Query | O(1) | Returns the component query used by the ECS runner to identify relevant entities. |

## Integration Patterns

### Standard Usage
This system is not used directly in gameplay code. Its behavior is configured externally through the server's gameplay configuration files, specifically by defining a ForgottenTempleConfig.

```yaml
# Example gameplay config entry (conceptual)
pluginConfigs:
  - com.hypixel.hytale.builtin.adventure.memories.temple.ForgottenTempleConfig:
      minYRespawn: -50.0
      respawnSoundIndex: "hytale:sfx.player.respawn_whoosh"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TempleRespawnPlayersSystem()`. The ECS framework is solely responsible for its lifecycle. Direct instantiation will result in a non-functional system that is not registered with the game loop.
-   **Manual Invocation:** Never call the `tick` method directly. This bypasses the ECS scheduler, the CommandBuffer synchronization, and the entity query filter, which will lead to unpredictable behavior and severe state corruption.

## Data Pipeline
The system processes data by reading from the world state and writing commands to a buffer for later execution.

> Flow:
> ECS Scheduler identifies player entities -> **TempleRespawnPlayersSystem.tick()** -> Reads TransformComponent Y-position -> Reads ForgottenTempleConfig -> If below threshold, writes Teleport component to CommandBuffer -> ECS Synchronization Point applies teleport -> Player position is updated.


---
description: Architectural reference for EntitySnapshotHistoryCommand
---

# EntitySnapshotHistoryCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class EntitySnapshotHistoryCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntitySnapshotHistoryCommand is a server-side administrative command designed for debugging and visualizing the server's entity state history. It functions as a concrete implementation within the server's command processing system, triggered by text input from an authorized user.

Architecturally, this command serves as a direct interface to the core Entity Component System (ECS), specifically targeting the server's rollback and state management capabilities. Its primary responsibility is to query for all entities possessing a SnapshotBuffer component. This component maintains a circular buffer of historical EntitySnapshot objects for a given entity, capturing its state at previous server ticks.

The command iterates through every stored snapshot for every relevant entity, extracts the historical position, and then uses the ParticleUtil to spawn a visual effect at that location. This creates a "trail" in the world, allowing administrators to visually inspect the movement history of entities, which is critical for debugging issues related to lag compensation, anti-cheat, and general physics.

## Lifecycle & Ownership
- **Creation:** A single instance of EntitySnapshotHistoryCommand is created by the server's command registration system during the server bootstrap phase. It is discovered, instantiated, and registered against the command name "history".
- **Scope:** The command object itself is a long-lived singleton, persisting for the entire server session within the command registry. However, its execution context is ephemeral; the `execute` method is invoked for a brief period in response to a specific user command and does not persist.
- **Destruction:** The object is garbage collected when the server shuts down and its containing command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable fields and all necessary data (World, EntityStore) is provided as arguments to the `execute` method. Its behavior is entirely determined by the state of the world at the time of execution.

- **Thread Safety:** This class is **not thread-safe** and must be executed on the main server thread. The `execute` method iterates over ECS data structures (`store.forEachChunk`) which are not designed for concurrent modification. Furthermore, the use of `SpatialResource.getThreadLocalReferenceList()` is a strong indicator of a thread-local object pool, which will cause race conditions and data corruption if accessed from multiple threads simultaneously. The server's command system is responsible for ensuring this command is invoked on the correct thread.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(C * E * H * P) | Executes the command logic. Iterates through all chunks (C) with entities (E) that have a snapshot history (H), and for each point, performs a spatial query for players (P). |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by the server's command handler when a privileged user types the corresponding command into the server console.

```
// User input in game console
/entity snapshot history
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EntitySnapshotHistoryCommand()`. The command is useless without being registered in the command system, and its `execute` method requires a valid `CommandContext` that is only built by that system.
- **Manual Invocation:** Do not call the `execute` method directly from other systems. This bypasses the command system's threading and permission models, likely leading to concurrency exceptions or data corruption. If you need to trigger this logic, dispatch it through the server's central command execution service.

## Data Pipeline
The data flow for this command is initiated by a user and results in a visual effect for nearby players.

> Flow:
> User Command String -> Command System Parser -> **EntitySnapshotHistoryCommand.execute** -> ECS Query (for SnapshotBuffer) -> Iteration over historical EntitySnapshot objects -> Spatial Query (for nearby players) -> ParticleUtil.spawnParticleEffect -> Network Packet to Client -> Particle Rendering


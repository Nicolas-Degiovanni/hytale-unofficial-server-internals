---
description: Architectural reference for ActionSpawnParticles
---

# ActionSpawnParticles

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient

## Definition
```java
// Signature
public class ActionSpawnParticles extends ActionBase {
```

## Architecture & Concepts
ActionSpawnParticles is a concrete implementation of the ActionBase contract within the server-side NPC Behavior System. Its sole responsibility is to command the creation of a particle effect at a location relative to its parent NPC. This class acts as a bridge between the high-level NPC logic (e.g., a behavior tree) and the low-level world rendering systems.

It operates as a "fire-and-forget" command. When executed, it calculates the final world-space coordinates for the effect, determines which players are close enough to perceive it, and dispatches a request to the ParticleUtil service. This design decouples NPC behavior from the complexities of particle replication and network optimization.

A key architectural feature is its use of a spatial query to identify nearby players. This ensures that particle effect data is only networked to relevant clients, significantly reducing network overhead in populated areas.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly with the *new* keyword. They are instantiated by the server's asset loading pipeline when an NPC's behavior configuration is loaded into memory. A corresponding BuilderActionSpawnParticles, typically defined in a JSON asset, provides the configuration data.
- **Scope:** The object's lifetime is bound to the NPC's loaded behavior definition (e.g., its Role). It is a reusable, immutable component that persists as long as the parent NPC's configuration is active.
- **Destruction:** The object is marked for garbage collection when the parent NPC's behavior definition is unloaded from memory, for instance, when an NPC is despawned and its associated assets are purged.

## Internal State & Concurrency
- **State:** **Immutable**. The core properties of the action—the particle system name, the positional offset, and the visibility range—are declared as final fields. They are initialized once during construction via the builder and cannot be modified thereafter.
- **Thread Safety:** The class is inherently thread-safe due to its immutable state. However, its methods, particularly **execute**, are designed for a specific, single-threaded execution context.

    **WARNING:** The execute method is not safe to call from arbitrary threads. It accesses and manipulates shared, non-thread-safe engine components like the EntityStore and relies on a ThreadLocal list for its spatial query results. It must only be invoked from the main server game loop thread.

## API Surface
The public contract is minimal, consisting only of the inherited execute method which is the primary entry point for the NPC behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(log N + M) | Triggers the particle effect. Complexity is dominated by the spatial query, where N is the total number of players and M is the number of players found within the effect's range. Always returns true. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly in Java code. Instead, they define its properties within an NPC's JSON asset file. The engine's behavior system then invokes the action when the NPC's state machine or behavior tree reaches the appropriate node.

The conceptual invocation by the engine looks like this:
```java
// This code is executed by a higher-level system like a Role or Behavior Tree.
// A developer does not write this.

// Assume 'currentAction' is an instance of ActionSpawnParticles
boolean actionCompleted = currentAction.execute(npcRef, npcRole, sensorInfo, deltaTime, entityStore);

if (actionCompleted) {
    // Transition to the next state or action
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ActionSpawnParticles()`. This bypasses the critical asset-building pipeline, resulting in an unconfigured and non-functional object. All actions must be defined in data assets.
- **Off-Thread Execution:** Calling the execute method from a worker thread will corrupt server state and cause unpredictable crashes, including ConcurrentModificationExceptions, due to unsafe access to the world's entity store.
- **State Caching:** Do not cache the return value of execute. It always returns true and its purpose is to trigger a side effect, not to provide a meaningful stateful result.

## Data Pipeline
The flow of data and control for this action begins with the NPC's logic and ends with a visual effect on a player's screen.

> Flow:
> NPC Behavior Tree Tick -> **ActionSpawnParticles.execute()** -> TransformComponent Query -> Spatial Player Query -> ParticleUtil.spawnParticleEffect() -> Network Subsystem -> Client Receives Packet -> Client-Side Particle System Renders Effect


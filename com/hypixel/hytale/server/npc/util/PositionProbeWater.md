---
description: Architectural reference for PositionProbeWater
---

# PositionProbeWater

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient Utility

## Definition
```java
// Signature
public class PositionProbeWater extends PositionProbeBase {
```

## Architecture & Concepts
PositionProbeWater is a specialized, stateful utility for spatial queries within the server's world simulation. Its primary function is to determine the physical state of an entity's bounding box at a specific position, with a specific focus on interactions with water.

This class extends the generic PositionProbeBase, inheriting its core collision detection logic. It operates as a *Strategy* pattern implementation, providing a custom `blockTest` method that overrides the parent's default block interaction rules. This specialization allows it to not only detect solid collisions but also to determine if an entity is submerged to a specific swimming depth.

It is a critical component for the server-side NPC AI and movement systems. These systems rely on probes like this to make decisions about pathfinding, physics, and animations. For example, an NPC controller would use PositionProbeWater to check if a prospective move would land it in water, allowing it to switch to a swimming state.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level system, typically an NPC movement or AI controller, immediately before a world query is needed. It is a lightweight, short-lived object.
- **Scope:** The object's lifetime is intended to be confined to the scope of a single method call. It is created, used once to probe a position, and then should be immediately discarded.
- **Destruction:** The object holds no external resources and is managed by the Java garbage collector. It becomes eligible for collection as soon as it falls out of scope.

## Internal State & Concurrency
- **State:** Highly mutable. The core purpose of this class is to mutate its internal fields (such as inWater, onGround, touchCeil) during the execution of the probePosition method. The state is only considered valid *after* a probe has been completed and *before* a subsequent call to reset or a new probe.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the server's main game tick. Accessing an instance from multiple threads will result in race conditions and unpredictable, corrupted state.

## API Surface
The public contract is focused exclusively on initiating the world query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| probePosition(...) | boolean | O(N) | Executes the spatial query. N is the number of blocks intersecting the bounding box. Mutates internal state to reflect collision and water status. |

## Integration Patterns

### Standard Usage
The class is designed to be instantiated, used for a single query, and then have its state inspected by the calling system.

```java
// An NPC AI system deciding its next move
PositionProbeWater waterProbe = new PositionProbeWater();
CollisionResult result = new CollisionResult();
Vector3d nextPosition = npc.calculateNextPosition();
Box npcBounds = npc.getBoundingBox();

// Execute the probe
waterProbe.probePosition(entityStoreRef, npcBounds, nextPosition, result, npc.getSwimDepth(), accessor);

// Make a decision based on the probe's resulting state
if (waterProbe.inWater) {
    npc.switchToSwimmingState();
} else if (waterProbe.onGround) {
    npc.switchToWalkingState();
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of PositionProbeWater in fields or persist them between game ticks. They are transient and their state is ephemeral.
- **Instance Reuse:** Avoid reusing a single instance for multiple probes without a deep understanding of its lifecycle. While the `reset` method exists, creating a new instance is safer and prevents subtle bugs from state leakage.
- **Concurrent Access:** Never pass an instance to another thread or access its fields while a `probePosition` call may be in flight on another thread.

## Data Pipeline
PositionProbeWater acts as a synchronous data transformer, converting a spatial query into a boolean state machine.

> Flow:
> NPC AI System -> **PositionProbeWater.probePosition**(position, bounds) -> World Block Storage Query -> **Internal State Mutation**(inWater, onGround) -> NPC AI System reads state


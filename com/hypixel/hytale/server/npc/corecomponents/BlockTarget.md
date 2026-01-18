---
description: Architectural reference for BlockTarget
---

# BlockTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Transient Component

## Definition
```java
// Signature
public class BlockTarget {
```

## Architecture & Concepts
The BlockTarget class is a stateful component that represents an NPC's specific interest in a single block within the game world. It is not merely a data container for a position; it is a fundamental part of the server's NPC resource management and task execution system.

This component acts as a handle for an NPC's **reservation** on a world resource. When an AI behavior, such as mining or harvesting, identifies a target block, it uses the server's resource management system (via the Blackboard) to "claim" that block. The BlockTarget instance holds the result of this claim, preventing other NPCs from targeting the same block simultaneously. This cooperative locking mechanism is essential for creating efficient and believable NPC group behaviors and preventing task conflicts.

The inclusion of a `chunkChangeRevision` field allows AI behaviors to perform cheap cache invalidation. If the world chunk containing the target block is modified, the revision number will change, signaling to the NPC that its target information may be stale and requires re-evaluation.

## Lifecycle & Ownership
- **Creation:** A BlockTarget instance is typically created as a member field within a higher-level AI component or directly within an NPCEntity. It is a plain Java object whose lifecycle is manually managed by its owner. It is not created via dependency injection or a central factory.

- **Scope:** The object's lifetime is bound to its owning NPC or AI behavior. It persists as long as the parent entity exists, but its internal state is transient, representing the NPC's *current* block-related task.

- **Destruction:** Explicit cleanup is managed via the `reset` method. This is a critical step. Failure to call `reset` when a task is completed, aborted, or the NPC is destroyed will result in a **resource leak**, where the reservation on the block is never released. This would render the block unusable by other NPCs indefinitely. The Java object itself is eventually garbage collected once its owner is reclaimed.

## Internal State & Concurrency
- **State:** The BlockTarget is highly mutable. Its fields are continuously updated by AI systems to reflect the status of a block-targeting task. It caches the target position, block type, and a reference to the resource reservation itself. The `isActive` method provides a view into this state, determining if the component currently represents a valid, active target.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPC's AI logic, which executes on a single, predictable thread within the server's main tick loop. Any concurrent modification from other threads without external locking will corrupt the component's state and, more critically, the global resource reservation system, leading to deadlocks or race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| reset(NPCEntity parent) | void | O(1) | **Critical Cleanup Method.** Releases the resource reservation held by this target and resets all internal fields to their default, inactive state. |
| isActive() | boolean | O(1) | Returns true if the target currently holds data for a valid block (where block type is a non-negative integer). |
| getPosition() | Vector3d | O(1) | Returns the world-space position of the target block. Returns a sentinel value if inactive. |
| setReservationHolder(ResourceView) | void | O(1) | Associates this target with a specific resource lock. This is typically called by the AI behavior that acquires the lock. |

## Integration Patterns

### Standard Usage
A typical AI behavior will use the BlockTarget as a state field. The pattern involves finding a block, reserving it, populating the BlockTarget, executing the task, and finally resetting it.

```java
// Within an AI Behavior class...
private final BlockTarget currentTarget = new BlockTarget();

public void findAndProcessBlock(NPCEntity self) {
    // 1. Find a suitable block and acquire a reservation
    Vector3d foundPos = findNearbyOre();
    ResourceView reservation = blackboard.getResourceManager().reserve(foundPos);

    if (reservation != null) {
        // 2. Populate the BlockTarget with reservation data
        this.currentTarget.getPosition().assign(foundPos);
        this.currentTarget.setFoundBlockType(World.getBlock(foundPos));
        this.currentTarget.setReservationHolder(reservation);

        // 3. Execute task (e.g., pathfind to currentTarget.getPosition())
        // ...
    }
}

public void onTaskFinished(NPCEntity self) {
    // 4. CRITICAL: Reset the target to release the reservation
    this.currentTarget.reset(self);
}
```

### Anti-Patterns (Do NOT do this)
- **Forgetting to Reset:** The most severe anti-pattern is failing to call `reset` upon task completion, failure, or entity death. This will cause the resource reservation to be held forever, effectively removing that block from the pool of available resources for all other NPCs.
- **State Sharing:** Do not share a single BlockTarget instance between multiple NPCs or even multiple concurrent behaviors within the same NPC. Each distinct block-targeting task requires its own dedicated BlockTarget instance.
- **External Modification:** Modifying the internal state of a BlockTarget (e.g., calling `setPosition`) without acquiring and associating a corresponding `ResourceView` reservation will break the contract of the resource management system. The data in the BlockTarget will not match the reality of the reservation system.

## Data Pipeline
BlockTarget acts as a stateful endpoint in the NPC task data flow, rather than a processing node.

> Flow:
> World Scanner (AI Behavior) -> Blackboard Resource Manager -> **BlockTarget** (State is populated) -> Pathfinding & Action Systems (State is read) -> **BlockTarget.reset()** -> Blackboard Resource Manager (Reservation is released)


---
description: Architectural reference for NPCFlockCommand
---

# NPCFlockCommand

**Package:** com.hypixel.hytale.server.flock.commands
**Type:** Utility

## Definition
```java
// Signature
public class NPCFlockCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The NPCFlockCommand class serves as a command dispatcher within the server's command processing system. It does not implement any command logic itself; instead, it functions as a container that groups multiple related sub-commands under a single `/flock` namespace. This pattern simplifies command registration and organizes player-facing interactions with the NPC flocking system.

The primary architectural significance of this class lies in its static helper methods, specifically `forNpcEntitiesInViewCone` and `anyEntityInViewCone`. These methods encapsulate a critical and reusable piece of game logic: querying for entities within a player's forward-facing view cone.

This view cone query is a two-stage process for performance:
1.  **Broad Phase:** A coarse, distance-based query is performed against a `SpatialResource`. This is a spatial partitioning data structure (e.g., an octree or grid) that rapidly identifies all entities within a given radius of the player, minimizing the number of entities to check.
2.  **Narrow Phase:** The results from the broad phase are then filtered more precisely. Each potential target's position is checked against the player's head rotation (yaw) using vector mathematics to determine if it falls within the defined view angle.

This approach avoids a naive iteration over all entities in the world, which would be computationally prohibitive.

### Lifecycle & Ownership
-   **Creation:** A single instance of NPCFlockCommand is created by the server's command registration system during the server bootstrap sequence. The constructor immediately registers its sub-commands (Join, Grab, Leave).
-   **Scope:** The object is a stateless singleton that persists for the entire lifetime of the server.
-   **Destruction:** The instance is discarded and garbage collected during server shutdown.

## Internal State & Concurrency
-   **State:** This class is stateless. It holds no mutable fields and its sole purpose is to route command execution and provide static utility functions. All state modifications occur within the Entity Component System via the `Store<EntityStore>` parameter passed into its methods.

-   **Thread Safety:** The static methods are not inherently thread-safe and are designed to be executed on the main server thread for a given world. The use of `SpatialResource.getThreadLocalReferenceList()` is a deliberate optimization to prevent list allocation overhead and avoid concurrency issues if the command system were to use a thread pool.

    **Warning:** Calling `forNpcEntitiesInViewCone` or `anyEntityInViewCone` from an asynchronous thread with a `Store` that is being actively modified by the main game loop will lead to undefined behavior, including `ConcurrentModificationException` or data corruption. All interactions with the ECS `Store` must be synchronized with the world's tick cycle.

## API Surface
The primary public contract consists of the static utility methods for spatial querying.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forNpcEntitiesInViewCone(player, store, predicate) | int | O(k log n) | Executes a predicate against all NPCEntity instances within the player's view cone. Returns the count of successful predicate tests. Complexity depends on the spatial index (n total entities) and local entity density (k entities in radius). |
| anyEntityInViewCone(player, store, predicate) | boolean | O(k log n) | Executes a predicate against entities in the player's view cone, short-circuiting and returning true on the first match. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or use. Its sub-commands are invoked by players via the chat console. The static helper methods, however, can be leveraged by other server-side systems to perform player-centric spatial queries.

```java
// Example of using the static helper from another server system
// to find NPCs the player is looking at.

int friendlyNpcsInView = NPCFlockCommand.forNpcEntitiesInViewCone(
    playerEntityRef,
    world.getStore(EntityStore.class),
    (npcRef, npcComponent) -> {
        // Predicate logic: return true if the NPC is friendly
        return npcComponent.getFaction() == Faction.FRIENDLY;
    }
);

if (friendlyNpcsInView > 0) {
    player.sendMessage("You see a friendly face!");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new NPCFlockCommand()`. The class is instantiated and managed exclusively by the server's command system.
-   **Stateful Predicates:** The predicates passed to the static helpers should be stateless. Relying on side effects within the predicate lambda can lead to unpredictable behavior, especially as the order of iteration is not guaranteed.
-   **Unsynchronized Access:** Do not call the static helper methods from a separate thread without first scheduling the task to run on the main world thread. Direct, concurrent access to the `Store` is unsafe.

## Data Pipeline
The data flow for a typical command execution is initiated by a player and results in a modification to the Entity Component System.

> Flow:
> Player Chat Input (`/flock grab`) -> Server Network Layer -> Command Parser -> **NPCFlockCommand** -> GrabCommand.execute() -> **forNpcEntitiesInViewCone()** -> SpatialResource Query -> ECS Store Access -> FlockMembership Component Added -> Message sent back to Player

---


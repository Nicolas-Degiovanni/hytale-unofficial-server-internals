---
description: Architectural reference for ResourceView
---

# ResourceView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.resource
**Type:** Transient State Object

## Definition
```java
// Signature
public class ResourceView extends BlockRegionView<ResourceView> {
```

## Architecture & Concepts
The ResourceView is a specialized, server-side component within the NPC AI's Blackboard system. Its primary function is to act as a concurrency control mechanism for world resources. It provides NPCs with a system to query and reserve individual blocks within a specific world region, preventing multiple NPCs from attempting to interact with the same resource (e.g., a tree, an ore vein) simultaneously.

Architecturally, this class implements a spatial reservation system. It maintains two core data structures for efficient lookups:
1.  A spatial index, `reservationsBySection`, which partitions the region into vertical sections for fast coordinate-based queries (Is this block reserved?).
2.  An entity-based index, `reservationsByEntity`, which maps an entity's unique reference to its reservation. This allows for fast cleanup when an entity completes its task or is removed from the world.

The methods `isOutdated` and `getUpdatedView` are notable for their trivial implementations (returning `false` and `this`, respectively). This indicates that, unlike other blackboard views which may need to be rebuilt when world state changes, the ResourceView is designed to be a long-lived, mutable state container. Its state is not derived from world data but is instead managed directly by the AI systems that use it.

## Lifecycle & Ownership
-   **Creation:** A ResourceView is instantiated directly by a higher-level manager responsible for a specific world region, likely a central Blackboard or a ViewManager. The `long index` passed during construction permanently associates this instance with that region.
-   **Scope:** The object persists as long as the world region it represents is loaded and actively managed by the server. Its lifetime is not tied to any single NPC but to the region itself.
-   **Destruction:** The class has no explicit destruction logic, as evidenced by the empty `cleanup` and `onWorldRemoved` methods. It is designed to be garbage collected when the managing system that holds the reference to it is destroyed.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful. The internal `HashMap` and `IntSet` arrays track the current reservation status of blocks within its region. This state is modified exclusively through the `reserveBlock` and `clearReservation` methods.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. The internal collections (`HashMap`, `IntOpenHashSet`) are not synchronized. All method calls that mutate state must be performed on the main server thread to prevent race conditions, data corruption, or `ConcurrentModificationException`. Unmanaged multi-threaded access will lead to system instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isBlockReserved(int x, int y, int z) | boolean | O(1) | Checks if a block at the given world coordinates is currently reserved. |
| reserveBlock(NPCEntity entity, int x, int y, int z) | void | O(1) | Creates a reservation for the specified block, associating it with the given entity. |
| clearReservation(Ref<EntityStore> ref) | void | O(1) | Removes the reservation associated with the given entity reference. |
| isOutdated(Ref, Store) | boolean | O(1) | Always returns false, indicating the view's state is managed internally, not derived from world state. |
| getUpdatedView(Ref, ComponentAccessor) | ResourceView | O(1) | Always returns `this`, reinforcing that the instance is long-lived and mutable. |

## Integration Patterns

### Standard Usage
The ResourceView is intended to be used by NPC behavior trees or finite state machines to coordinate access to limited world resources. The standard interaction is a check-then-act-then-clear pattern.

```java
// Within an NPC's update logic, context provides access to the region's blackboard
ResourceView resourceView = blackboard.getView(ResourceView.class);
BlockPos targetBlock = findNearbyResource();

// 1. Check if the resource is available
if (!resourceView.isBlockReserved(targetBlock.getX(), targetBlock.getY(), targetBlock.getZ())) {
    // 2. Reserve the resource before acting
    resourceView.reserveBlock(this.getNPCEntity(), targetBlock.getX(), targetBlock.getY(), targetBlock.getZ());
    
    // ... proceed with pathfinding and interaction logic ...
    
    // 3. CRITICAL: Clear the reservation upon completion or failure
    resourceView.clearReservation(this.getNPCEntity().getReference());
}
```

### Anti-Patterns (Do NOT do this)
-   **Leaked Reservations:** Failing to call `clearReservation` after an NPC has finished its task, been interrupted, or died. This will cause the block to be permanently reserved, effectively removing it from the pool of available resources for all other NPCs.
-   **Multi-threaded Access:** Accessing a ResourceView instance from any thread other than the main server tick thread. This will cause unpredictable behavior and crashes.
-   **External Instantiation:** AI behaviors should not create their own `new ResourceView()`. They must always retrieve the shared, authoritative instance for the region from the Blackboard system.


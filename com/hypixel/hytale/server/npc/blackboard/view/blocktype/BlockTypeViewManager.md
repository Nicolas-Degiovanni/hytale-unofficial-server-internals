---
description: Architectural reference for BlockTypeViewManager
---

# BlockTypeViewManager

**Package:** com.hypixel.hytale.server.npc.blackboard.view.blocktype
**Type:** Service / Manager

## Definition
```java
// Signature
public class BlockTypeViewManager extends BlockRegionViewManager<BlockTypeView> {
```

## Architecture & Concepts
The BlockTypeViewManager is a specialized factory and lifecycle manager for BlockTypeView instances within the server's AI Blackboard system. It does not manage views directly; instead, it provides a concrete implementation strategy for its parent, the generic BlockRegionViewManager.

Its primary architectural role is to decouple the generic logic of managing region-based data views from the specific logic required for views concerning block types. The parent class handles the core mechanics of caching, retrieving, and cleaning up views based on a spatial index. This class provides the "how" for two critical operations:
1.  **Creation:** It defines how to construct a new BlockTypeView when the parent manager receives a request for a region that is not yet cached.
2.  **Cleanup:** It provides the predicate that determines when a specific BlockTypeView is no longer needed and can be garbage collected.

This class also instantiates and owns a BlockPositionEntryGenerator, which it injects into every BlockTypeView it creates. This ensures that all views managed by this instance share the same underlying generator, likely for consistency or performance.

### Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's central Blackboard service during its initialization phase, typically at world load. It is registered as a specific type of view manager.
-   **Scope:** The BlockTypeViewManager is a session-scoped singleton. It persists for the entire lifetime of the server world session. Its purpose is to manage the lifecycle of the more transient BlockTypeView objects.
-   **Destruction:** The instance is discarded and eligible for garbage collection only when the world is shut down and the parent Blackboard service is decommissioned.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its only field, the BlockPositionEntryGenerator, is instantiated once and is immutable from the perspective of this class. The true state—the collection of active BlockTypeView instances—is maintained and guarded by the parent BlockRegionViewManager.
-   **Thread Safety:** This class is thread-safe by design, as its methods are re-entrant and do not modify shared state. However, it operates within a highly concurrent environment. The parent BlockRegionViewManager is responsible for ensuring that calls to `createView` and `shouldCleanup` are synchronized and that the underlying view collection is modified atomically. External systems must not assume that the views managed by this class are safe to access from multiple threads without proper synchronization on the view object itself.

## API Surface
The public API is inherited from BlockRegionViewManager. The methods defined in this class are `protected` and represent the implementation of the parent's abstract contract. They are not intended for direct external invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createView(index, blackboard) | BlockTypeView | O(1) | **Framework Internal.** Factory method invoked by the parent manager to instantiate a new view for a given region index. |
| shouldCleanup(view) | boolean | O(1) | **Framework Internal.** Predicate invoked by the parent manager's cleanup task. Returns true if the view is considered derelict and can be destroyed. |

## Integration Patterns

### Standard Usage
Developers do not interact with the BlockTypeViewManager directly. Instead, they request a view from the high-level Blackboard, which delegates the request to the appropriate manager. The manager's operation is transparent to the end-user.

```java
// An AI entity's perspective. The manager is never seen.
Blackboard blackboard = entity.getBlackboard();

// Requesting the view triggers the BlockTypeViewManager internally
// if a new view needs to be created.
BlockTypeView view = blackboard.getView(BlockTypeView.class, entity.getRegionIndex());

// Use the view to query for data
Set<BlockPosition> waterBlocks = view.getBlocks(BlockType.WATER);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never construct this class with `new BlockTypeViewManager()`. It is a system-level service that must be instantiated and managed by the core Blackboard system to function correctly.
-   **External Lifecycle Management:** Do not attempt to manually control the lifecycle of the BlockTypeView objects it produces. The manager's entire purpose is to automatically clean up views when they are no longer in use. Holding a permanent reference to a view may create a memory leak by preventing `shouldCleanup` from ever returning true.

## Data Pipeline
The manager acts as a factory and a policy-maker within two distinct data flows managed by its parent.

**View Creation Flow:**
> AI System Request for View -> Blackboard.getView() -> BlockRegionViewManager -> **BlockTypeViewManager.createView()** -> New BlockTypeView returned to AI

**View Cleanup Flow:**
> BlockRegionViewManager (Periodic Cleanup Task) -> For each cached view -> **BlockTypeViewManager.shouldCleanup(view)** -> If true, view is removed from cache


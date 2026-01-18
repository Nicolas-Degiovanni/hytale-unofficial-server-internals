---
description: Architectural reference for ResourceViewManager
---

# ResourceViewManager

**Package:** com.hypixel.hytale.server.npc.blackboard.view.resource
**Type:** Stateful Manager

## Definition
```java
// Signature
public class ResourceViewManager extends BlockRegionViewManager<ResourceView> {
```

## Architecture & Concepts
The ResourceViewManager is a specialized component within the server-side NPC AI framework. It inherits from the generic BlockRegionViewManager, narrowing its focus to the management of **ResourceView** objects. Its primary role is to act as a centralized factory and cache for shared representations of world resources, such as trees, ore veins, or other harvestable items.

This manager is a critical piece of the AI's **Blackboard** system, a common design pattern for sharing state and data between different AI behaviors. By providing managed, shared views of resources, it prevents common AI conflicts, such as multiple NPCs attempting to harvest the same resource simultaneously. It achieves this by managing a lifecycle tied to "reservations," where an NPC must formally reserve a resource through its view before acting upon it.

In essence, this class provides the concrete implementation details—the factory method and the cleanup policy—for a more abstract, spatially-indexed view management system provided by its parent.

## Lifecycle & Ownership
- **Creation:** The ResourceViewManager is not intended for direct instantiation. It is created and owned by a higher-level system, typically the server's core AI service or as part of a world's Blackboard initialization. There is likely one instance per logical AI context.
- **Scope:** Its lifetime is coupled to its owner. It persists as long as the parent AI system is active for a given world or server session.
- **Destruction:** The manager is eligible for garbage collection when its owning system is shut down. The internal ResourceView objects it manages have a more dynamic lifecycle; they are periodically pruned by a cleanup process within the parent BlockRegionViewManager, which uses the **shouldCleanup** method as its decision-making predicate.

## Internal State & Concurrency
- **State:** This class itself is stateless; its logic is defined by its methods. However, its parent class, BlockRegionViewManager, is highly stateful. It maintains an internal collection, likely a map or spatial hash, that associates region indices with their corresponding ResourceView instances. This collection is mutable, serving as a live cache of resource states for the AI system.
- **Thread Safety:** **WARNING:** This component operates in a multi-agent environment and must be thread-safe. The parent BlockRegionViewManager is expected to implement synchronization controls (e.g., locks, concurrent collections) to handle simultaneous access from different NPC behavior threads. All interactions with the manager's view cache must be atomic to prevent race conditions during view creation, retrieval, and cleanup.

## API Surface
The public API is inherited from BlockRegionViewManager. The methods defined here are protected and part of a template method pattern, dictating the specific behavior of this manager type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createView(long, Blackboard) | ResourceView | O(1) | **Framework Hook.** Factory method invoked by the parent manager to instantiate a new ResourceView for a given region index when no cached view exists. |
| shouldCleanup(ResourceView) | boolean | O(N) | **Framework Hook.** Predicate used by the parent's cleanup process. Returns true if a view has no active entity reservations, marking it for eviction from the cache. Complexity is proportional to the number of reservations (N). |

## Integration Patterns

### Standard Usage
Direct interaction is uncommon. Developers typically interact with a higher-level AI service or Blackboard API that uses this manager under the hood to acquire a view of a resource.

```java
// Hypothetical usage from within an NPC Behavior
// The manager is retrieved from a central service, not instantiated.
Blackboard blackboard = npc.getBlackboard();
ResourceViewManager resourceManager = blackboard.getViewManager(ResourceViewManager.class);

// The manager's public API (from the parent class) is used to get a view
long resourceRegionIndex = world.getRegionIndex(resourcePosition);
ResourceView view = resourceManager.getOrCreateView(resourceRegionIndex);

// The NPC then interacts with the view to reserve the resource
if (view.reserve(npc.getEntityId())) {
    // Proceed with harvesting behavior...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new ResourceViewManager()**. The manager is tightly coupled to the lifecycle of the server's AI framework and must be retrieved from the appropriate context or service registry.
- **State Assumption:** Do not assume a returned ResourceView will exist indefinitely. If it has no reservations, it can be garbage collected at any time by the manager's cleanup process. Always re-acquire or validate views when beginning a new behavior.
- **Bypassing Reservations:** Interacting with a resource without first securing a reservation through its ResourceView will break the coordination mechanism and lead to undefined AI behavior and resource contention.

## Data Pipeline
This manager acts as a gatekeeper and cache in the flow of information from the world state to an NPC's decision-making logic.

> Flow:
> NPC Perception System (detects a resource) -> AI Behavior Tree (requests a shared view for the resource's region) -> **ResourceViewManager** (returns a cached view or creates a new one) -> NPC Behavior Tree (interacts with the ResourceView to reserve it) -> World State Update (NPC harvests the resource)


---
description: Architectural reference for IBlackboardViewManager
---

# IBlackboardViewManager

**Package:** com.hypixel.hytale.server.npc.blackboard.view
**Type:** Service Contract / Manager Interface

## Definition
```java
// Signature
public interface IBlackboardViewManager<View extends IBlackboardView<View>> {
```

## Architecture & Concepts
The **IBlackboardViewManager** interface defines the contract for a critical component within the server-side NPC Artificial Intelligence framework. Its primary responsibility is to act as a centralized factory and cache for **IBlackboardView** instances.

In Hytale's AI, a **Blackboard** is a shared data repository used by various AI behaviors to communicate and store state about a specific NPC. An **IBlackboardView** provides a structured, type-safe API to access and manipulate the data within a **Blackboard**.

This manager serves as the sole entry point for obtaining these views. It decouples AI systems from the direct creation and lifecycle management of views, ensuring that for any given entity, only one view instance exists. This prevents state desynchronization and race conditions. The manager abstracts the underlying storage mechanism, which maps an entity identifier (such as its ID or world position) to its corresponding AI state view.

This component is fundamental for the modularity of the AI system, allowing different behaviors to operate on a shared, consistent state without needing direct references to each other.

### Lifecycle & Ownership
- **Creation:** The concrete implementation of this interface is instantiated by the server's service container when a game world is initialized. It is not intended for manual creation.
- **Scope:** The manager's lifecycle is tightly bound to a specific game world. It persists for the entire duration that the world is loaded and active. The presence of the **onWorldRemoved** method confirms this world-scoped lifecycle.
- **Destruction:** The manager and all the views it contains are destroyed when the corresponding game world is unloaded. The **cleanup** and **onWorldRemoved** methods are the designated hooks for releasing all managed resources and breaking reference cycles.

## Internal State & Concurrency
- **State:** Any implementation of this interface is inherently stateful. It must maintain an internal collection, typically a map, that associates entity identifiers with their active **IBlackboardView** instances. This collection functions as a cache to ensure view instances are reused.
- **Thread Safety:** **CRITICAL:** Implementations must be thread-safe. Server-side systems, including AI, can operate across multiple threads. Access to the internal view cache must be synchronized to prevent concurrent modification exceptions and data corruption. This is typically achieved using concurrent collections or explicit locking mechanisms. The **forEachView** method, which accepts a **Consumer**, is a pattern designed for safe iteration over the managed views without exposing the underlying collection.

## API Surface
The public contract focuses on the acquisition, iteration, and cleanup of views.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(...) | View | O(1) | Retrieves or creates a view for an entity. Multiple overloads exist for different identifier types (Ref, Vector3d, ID). This is the primary factory method. |
| getIfExists(long) | View | O(1) | Retrieves a view only if it already exists in the cache. Returns null otherwise. This is a performance-critical method to avoid unnecessary allocations. |
| cleanup() | void | O(N) | Signals the manager to perform internal maintenance or cleanup on all managed views. |
| onWorldRemoved() | void | O(N) | The primary lifecycle hook called during world unloading. Responsible for the complete destruction of all views and releasing resources. |
| forEachView(Consumer) | void | O(N) | Safely executes a given operation on every active view managed by this instance. |
| clear() | void | O(N) | Immediately removes all views from the manager. This is a destructive operation typically used for hard resets. |

## Integration Patterns

### Standard Usage
AI behavior systems should always retrieve the world's **IBlackboardViewManager** instance from the server's service context. They then use it to get a view for the specific NPC they are processing.

```java
// An AI system retrieves the manager from the world context
IBlackboardViewManager<MyView> viewManager = world.getService(IBlackboardViewManager.class);

// It then gets a view for a specific entity ID and its blackboard
long entityId = ...;
Blackboard blackboard = ...;
MyView view = viewManager.get(entityId, blackboard);

// The system can now safely interact with the NPC's AI state
view.setTargetLocation(newDestination);
```

### Anti-Patterns (Do NOT do this)
- **Implementation Dependency:** Do not write code that depends on a concrete implementation of this interface. Always code against the **IBlackboardViewManager** interface to maintain modularity.
- **View Caching:** Do not cache the returned **View** objects in other systems across multiple game ticks. The view or the underlying entity may become invalid. Always request the view from the manager at the beginning of an operation.
- **Lifecycle Mismanagement:** Failure to call **onWorldRemoved** during server shutdown or world unloading will result in significant memory leaks, as all cached views and their associated blackboards will be orphaned.

## Data Pipeline
This component acts as a controller and cache rather than a linear data processor. Its primary flow is request-based.

> Flow:
> AI Behavior System -> **IBlackboardViewManager.get(entityId)** -> (Cache Hit/Miss) -> View Instance Returned -> AI Behavior System Reads/Writes to View -> Data updated in underlying **Blackboard**


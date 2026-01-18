---
description: Architectural reference for EntityTrackerSystems
---

# EntityTrackerSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.tracker
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class EntityTrackerSystems {
    // Contains only static fields and nested static classes.
    // This class is not designed to be instantiated.
}
```

## Architecture & Concepts
EntityTrackerSystems is a foundational part of the server's networking and world simulation architecture. It is not a single system, but rather a namespace that groups together a collection of highly-specialized Entity Component Systems (ECS) and associated Components. Collectively, these elements form the server's **visibility and replication layer**.

The primary responsibility of this subsystem is to determine which entities are visible to which players (or other "viewer" entities) on a per-tick basis. It then orchestrates the process of collecting state changes from those visible entities and serializing them into network packets for clients.

This system acts as the bridge between the in-memory game state (represented by ECS Components) and the serialized network protocol. It is the engine that drives nearly all entity-related network traffic, ensuring that clients only receive updates for entities within their view distance and relevance scope.

The core components of this architecture are:
- **EntityViewer Component:** Attached to entities that can "see" the world, such as players. It defines the view radius and acts as a buffer for outgoing network updates for that specific viewer.
- **Visible Component:** Dynamically attached to any entity that is currently being observed by at least one EntityViewer. It tracks which viewers can see it, enabling efficient state change detection.
- **Processing Pipeline:** A multi-stage pipeline of systems, ordered by the ECS scheduler, that executes each tick to calculate visibility, detect changes, queue updates, and send packets.

## Lifecycle & Ownership
The EntityTrackerSystems class itself is a static container and has no lifecycle. However, the systems and components it defines are managed by the ECS framework.

- **Creation:** The various nested system classes (e.g., CollectVisible, SendPackets) are instantiated once by the ECS framework during server bootstrap or world initialization. They are registered with the ECS scheduler which controls their execution. The components (EntityViewer, Visible) are created and attached to entities dynamically during gameplay.
- **Scope:** The systems persist for the lifetime of the server or world they are registered with. The components exist only as long as their host entity exists and the required conditions are met (e.g., a player is online, an entity is visible).
- **Destruction:** The systems are destroyed when the server or world shuts down. The components are destroyed along with their host entities. The RemoveEmptyVisibleComponent system is responsible for cleaning up the Visible component from entities that are no longer seen by any viewer, preventing component bloat.

## Internal State & Concurrency
- **State:** The state is not held within the EntityTrackerSystems class itself, but within the components it defines.
    - **EntityViewer:** Highly mutable. Its internal collections (visible, updates, sent) are modified continuously throughout the tick by multiple systems. It acts as a per-viewer state buffer.
    - **Visible:** Highly mutable. Its maps of viewers (`visibleTo`, `previousVisibleTo`) are cleared and repopulated each tick. This component uses a double-buffering pattern to track which viewers were visible last tick versus this tick, allowing for efficient detection of newly visible entities.
- **Thread Safety:** This subsystem is designed for high-performance, parallel execution.
    - The ECS scheduler runs many of these systems in parallel across multiple threads. The `isParallel` method in each system signals this capability.
    - The Visible component uses a `StampedLock` to protect its maps during parallel updates from the AddToVisible system.
    - The EntityViewer component uses a `ConcurrentHashMap` for its `updates` map, allowing different systems to queue updates for the same viewer concurrently without explicit locking.
    - The SendPackets system utilizes a `ThreadLocal` list to avoid heap allocations and contention when building lists of removed entities in a parallel context.

    **WARNING:** Direct, unsynchronized access to the internal collections of EntityViewer or Visible components from outside this system's managed pipeline is extremely hazardous and will lead to race conditions and server instability.

## API Surface
The public API consists of static utility methods and system group definitions, not instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FIND_VISIBLE_ENTITIES_GROUP | SystemGroup | N/A | Defines the ECS execution phase for systems that calculate entity visibility. |
| QUEUE_UPDATE_GROUP | SystemGroup | N/A | Defines the ECS execution phase for systems that queue component updates based on visibility. |
| despawnAll(viewerRef, store) | boolean | O(N) | Forces a despawn of all entities currently visible to a specific viewer, sending a single removal packet. N is the number of visible entities. |
| clear(viewerRef, store) | boolean | O(N) | Clears the internal tracking state for a viewer without sending network packets. Used for hard resets. N is the number of visible entities. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. Its functionality is consumed implicitly by the ECS scheduler. A gameplay programmer's primary interaction is attaching and configuring the EntityViewer component to a player entity.

```java
// Attaching an EntityViewer component to a player entity upon login.
// The tracking systems will automatically pick up and manage this entity.

int viewDistanceInBlocks = 128;
IPacketReceiver playerConnection = player.getPacketReceiver();

commandBuffer.addComponent(playerEntityRef, new EntityViewer(viewDistanceInBlocks, playerConnection));
```

### Anti-Patterns (Do NOT do this)
- **Manual System Execution:** Do not instantiate and call the `tick` method on systems like `CollectVisible` or `SendPackets` manually. They are designed to be driven exclusively by the ECS scheduler, which respects their declared dependencies and ordering.
- **Modifying Internal State:** Do not directly add or remove entity references from an `EntityViewer.visible` set or a `Visible.visibleTo` map. This will bypass the tracking logic, corrupt the system's state, and lead to network desynchronization.
- **Reusing EntityViewer:** Do not share a single `EntityViewer` component instance across multiple entities. Each viewer must have its own unique instance to manage its state independently.

## Data Pipeline
The entity tracking process is a sequential pipeline executed every server tick. The flow is orchestrated by dependencies between systems and execution within specific SystemGroups.

> Flow:
> **Tick Start** ->
> **1. Cleanup Phase** (ClearEntityViewers, ClearPreviouslyVisible): Reset state from the previous tick. The `visible` set on each viewer is cleared, and the `visibleTo` maps on entities are swapped for double-buffering. ->
> **2. Visibility Calculation Phase** (FIND_VISIBLE_ENTITIES_GROUP): The CollectVisible system uses a spatial data structure to perform a proximity search around each EntityViewer, populating its `visible` set with nearby entities. ->
> **3. State Reconciliation Phase** (EnsureVisibleComponent, AddToVisible): The system ensures all entities in a viewer's `visible` set have a `Visible` component. It then updates the `Visible` component on each entity, registering the viewers that can now see it and identifying which are *newly* visible this tick. ->
> **4. Update Queuing Phase** (QUEUE_UPDATE_GROUP): Systems like EffectControllerSystem run. They inspect the `newlyVisibleTo` and `visibleTo` maps on entities they manage. They queue full-state updates for newly visible viewers and delta-updates for existing viewers. Updates are written into the `EntityViewer.updates` map. ->
> **5. Packet Serialization Phase** (SEND_PACKET_GROUP): The SendPackets system iterates over every EntityViewer. It compares the current `visible` set with the previously `sent` entities to determine which entities need to be despawned. It then bundles all despawns and all queued component updates into a single `EntityUpdates` packet and writes it to the viewer's network connection. ->
> **Tick End**


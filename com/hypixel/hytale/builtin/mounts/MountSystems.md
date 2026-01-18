---
description: Architectural reference for MountSystems
---

# MountSystems

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Utility

## Definition
```java
// Signature
public class MountSystems {
    // Contains multiple static inner classes representing ECS Systems
}
```

## Architecture & Concepts

MountSystems is not a single, instantiable object but rather a static container class that groups together a suite of related ECS (Entity Component System) systems. Collectively, these systems implement the entire server-side logic for entity mounting, covering both entity-on-entity and entity-on-block scenarios (e.g., a player riding a horse vs. sitting on a chair).

This class acts as a central hub for the "Mounting" feature, demonstrating a common pattern in Hytale's ECS architecture where a complex feature is decomposed into several small, single-responsibility systems. Each nested class targets a specific aspect of the mounting lifecycle:

*   **State Management:** Systems like **TrackedMounted** react to the addition or removal of the **MountedComponent**, ensuring the relationship between the rider and the mount is correctly established or torn down. It maintains data integrity by automatically updating the corresponding **MountedByComponent** on the mount.
*   **Input Processing:** The **HandleMountInput** system intercepts player input for a mounted entity and retargets movement commands to the mount itself, effectively allowing the player to control the mount.
*   **Lifecycle & Cleanup:** A series of reactive systems (**MountedEntityDeath**, **TeleportMountedEntity**, **RemoveMounted**, **RemoveMountedBy**) handle edge cases and cleanup. They ensure that an entity is always dismounted if it dies, teleports, or is otherwise removed from the world, preventing orphaned data and inconsistent game states.
*   **Networking:** The **TrackerUpdate** system is responsible for synchronizing the mount state with clients. It observes changes to the **MountedComponent** and queues network packets (**MountedUpdate**) to be sent to all relevant players, ensuring they see the entity correctly attached to its mount.
*   **Gameplay Logic:** Specialized systems like **OnMinecartHit** implement specific gameplay mechanics for mountable entities, such as a minecart breaking after being hit three times.

The entire design relies on declarative component manipulation. A developer does not call these systems directly; instead, they add or remove a **MountedComponent** from an entity, and this collection of systems automatically reacts to orchestrate the required logic.

### Lifecycle & Ownership

-   **Creation:** The MountSystems class itself is never instantiated. Its nested system classes are discovered and instantiated by the server's core ECS engine during the server bootstrap process.
-   **Scope:** Each individual system (e.g., HandleMountInput, TrackedMounted) is a singleton managed by the ECS engine. They persist for the entire lifetime of the server session.
-   **Destruction:** The systems are destroyed when the server shuts down. There is no manual cleanup required.

## Internal State & Concurrency

-   **State:** The MountSystems container is stateless. The individual nested systems are also designed to be stateless. They hold no per-entity data themselves; all state is read from and written to components within the ECS world via the **ArchetypeChunk** and **CommandBuffer** provided in their execution methods.
-   **Thread Safety:** These systems are thread-safe within the context of the Hytale ECS framework. The engine guarantees that systems are executed in a deterministic order defined by their dependencies. All state mutations are funneled through a **CommandBuffer**, which queues changes to be applied at a safe synchronization point at the end of a tick. This deferred mutation pattern prevents race conditions and ensures that systems operate on a consistent snapshot of the world state during their execution.

**WARNING:** Directly modifying components outside of a CommandBuffer from these systems would break thread safety and lead to unpredictable behavior.

## API Surface

The public contract of MountSystems is not a set of methods, but the collection of system classes it provides for dependency management and the components it operates on. The primary point of interaction for external code is the **MountedComponent**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HandleMountInput | System | O(N) | Ticking system. Processes player input for mounted entities and applies it to the mount. |
| TrackedMounted | System | O(1) | Reactive system. Manages the bidirectional link between rider and mount when a MountedComponent is added or removed. |
| MountedEntityDeath | System | O(1) | Reactive system. Forces a dismount when a mounted entity receives a DeathComponent. |
| TeleportMountedEntity | System | O(1) | Reactive system. Forces a dismount before an entity is teleported to prevent logical inconsistencies. |
| RemoveMounted / RemoveMountedBy | System | O(N) | Reactive cleanup systems. Ensure all passengers are dismounted when a mount is removed from the world. |
| TrackerUpdate | System | O(N) | Ticking system. Detects changes in MountedComponent and queues network packets for clients. |
| OnMinecartHit | System | O(1) | Reactive gameplay system. Handles damage logic specific to Minecart entities. |

## Integration Patterns

### Standard Usage

To make an entity ride another, one must add a **MountedComponent** to the riding entity. The systems within MountSystems will handle the rest. This is typically done within another system's logic via a CommandBuffer.

```java
// Example: A system that makes an entity ride another upon interaction.
// Assume 'riderRef' and 'mountRef' are valid entity references.

MountedComponent mountedComponent = new MountedComponent(mountRef, MountController.EntityMount, attachmentOffset);
commandBuffer.addComponent(riderRef, MountedComponent.getComponentType(), mountedComponent);

// The MountSystems.TrackedMounted system will now automatically execute,
// adding a MountedByComponent to the mountRef entity.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not create instances of the system classes (e.g., `new HandleMountInput()`). The ECS engine is solely responsible for their lifecycle.
-   **Manual State Management:** Do not manually add a **MountedByComponent** to a mount. The **TrackedMounted** system is the source of truth for this relationship and will add it automatically. Manually adding it can lead to a desynchronized state.
-   **Ignoring the CommandBuffer:** Modifying a **MountedComponent** directly on an entity's store bypasses the reactive nature of the systems and can cause network updates to be missed. All changes must go through the CommandBuffer.
-   **Incorrect Dependency Ordering:** If creating a custom system that interacts with mounts, ensure it declares a proper dependency (e.g., `Order.AFTER, TrackedMounted.class`) to avoid race conditions within a single tick.

## Data Pipeline

The systems form a reactive data pipeline triggered by component changes. The most common flow is initiating a mount and seeing it replicated to clients.

> Flow:
> External System Logic -> `commandBuffer.addComponent(rider, MountedComponent)` -> **TrackedMounted** System (adds `MountedByComponent` to mount) -> **PlayerMount** System (updates rider's `PlayerInput` state) -> **TrackerUpdate** System (detects network-dirty `MountedComponent`) -> Queues `MountedUpdate` network packet -> Client Receives Packet -> Client-side logic attaches entities visually.


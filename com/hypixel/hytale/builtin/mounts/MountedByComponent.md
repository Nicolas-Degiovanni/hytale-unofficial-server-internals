---
description: Architectural reference for MountedByComponent
---

# MountedByComponent

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Component (Data)

## Definition
```java
// Signature
public class MountedByComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The MountedByComponent is a fundamental data component within Hytale's Entity-Component-System (ECS) framework, specifically for the Mounts system. It serves a single, critical purpose: to track which entities are currently riding another entity.

This component is attached to the entity that **is being ridden** (the mount). It holds a list of references to its passengers. It is a pure data container; all logic for mounting, dismounting, and synchronizing passenger positions is handled by a corresponding *System*, such as a MountSystem, which reads from and writes to this component.

The use of Ref<EntityStore> is a key architectural choice. This is a non-owning, "weak" reference to another entity. This design prevents strong reference cycles between entities and ensures that if a passenger entity is destroyed, it does not leave a dangling, invalid pointer in the mount's state. The component's internal mechanisms are responsible for periodically cleaning up these invalid references.

### Lifecycle & Ownership
-   **Creation:** This component is instantiated and managed exclusively by the ECS framework. It is typically added to an entity when game logic dictates that the entity can be mounted, either through world data definitions or dynamically by a game system. Manual instantiation is an anti-pattern.
-   **Scope:** The lifecycle of a MountedByComponent is strictly bound to the lifecycle of its parent entity. It exists only as long as the entity it is attached to exists in the world.
-   **Destruction:** The component is destroyed automatically when its parent entity is removed from the world. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The component's state is mutable. Its core is a list of passenger references which is frequently modified as entities mount and dismount. The state represents a point-in-time truth of the mount-passenger relationship.

-   **Thread Safety:** **This component is not thread-safe.** All operations on an instance of MountedByComponent must be performed on the main game-loop thread. The internal data structure, ObjectArrayList, is not synchronized. Unmanaged access from other threads, such as network or physics threads, will result in state corruption or a ConcurrentModificationException. All interactions must be scheduled as tasks on the primary world thread.

## API Surface
The public API provides the essential primitives for managing the passenger list.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type definition for this component. |
| getPassengers() | List<Ref<EntityStore>> | O(N) | Returns the list of valid passengers. **Warning:** This method performs a linear scan to remove invalid references on every call, impacting performance on mounts with many passengers. |
| addPassenger(Ref) | void | O(1) amortized | Adds a new passenger entity to the internal list. |
| removePassenger(Ref) | void | O(N) | Removes a specific passenger from the list. Requires a linear scan. |
| clone() | Component | O(1) | Creates a new, **empty** instance. **Warning:** This method does not copy the passenger list. It is used by the ECS framework for creating component templates, not for duplicating entity state. |

## Integration Patterns

### Standard Usage
Interaction with this component should always be mediated by the ECS framework. A System retrieves the component from an entity to read or modify its state.

```java
// Executed within a System that has access to the entity
// Assume 'mountEntity' is the entity being ridden
// Assume 'riderRef' is a valid Ref<EntityStore> to the passenger

MountedByComponent mountedBy = mountEntity.getComponent(MountedByComponent.class);

// Check for null in case the entity is not a mount
if (mountedBy != null) {
    mountedBy.addPassenger(riderRef);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new MountedByComponent()`. The ECS framework is responsible for creating and attaching components to entities. Direct creation bypasses the engine's management and will lead to unpredictable behavior.
-   **Caching Passenger List:** Do not retrieve the passenger list via getPassengers and store it for an extended period. The list is live and can be modified by other systems. Furthermore, holding a reference to the list prevents the garbage collector from cleaning up invalid references if the original mount is destroyed. Always re-fetch the component and the list when needed.
-   **Cross-Thread Modification:** Never call addPassenger or removePassenger from any thread other than the main world update thread. This will corrupt the component's state and cause severe stability issues.

## Data Pipeline
The MountedByComponent acts as a state repository, not a data processor. It is a destination for commands from game systems and a source of truth for replication and rendering systems.

> **Flow (Player Mounts an Entity):**
> Player Input -> Network Packet -> Server-side MountSystem -> **MountedByComponent.addPassenger()** -> Entity State Change -> World State Replicator -> Client-side Entity Update -> Visual Attachment in Renderer


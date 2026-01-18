---
description: Architectural reference for NPCMountComponent
---

# NPCMountComponent

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Transient Data Component

## Definition
```java
// Signature
public class NPCMountComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The NPCMountComponent is a data-only component within the server-side Entity Component System (ECS) framework. It does not contain any logic. Instead, it serves as a state container that, when attached to an NPC entity, grants it the properties and data necessary to function as a player-ridable mount.

Its primary role is to store runtime state, such as the current owner and a home anchor position. Systems, such as a hypothetical MountSystem, query for entities that possess this component and execute the logic that governs mount behavior (e.g., player mounting, tethering, following).

The presence of a static CODEC field is a critical architectural feature. It signifies that this component is fully integrated into the engine's serialization and persistence pipeline. The engine uses this codec to automatically handle network replication to clients and for saving the component's state to disk during a world save.

## Lifecycle & Ownership
- **Creation:** An NPCMountComponent is not instantiated directly. It is added to an entity by a governing system, typically in response to a game event like a player taming an NPC or a world script designating an entity as a mount. The component is then managed by the entity's parent EntityStore.
- **Scope:** The lifecycle of this component is strictly bound to the lifecycle of the entity to which it is attached. It persists as long as the entity exists in the game world.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world, either through gameplay (e.g., death) or administrative action (e.g., despawning).

## Internal State & Concurrency
- **State:** The component's state is entirely mutable. Its fields, such as ownerPlayerRef and the anchor coordinates, are designed to be frequently updated by game systems during a server tick. It is a pure state-bag and performs no internal caching.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by systems operating on the main server game thread. Unsynchronized, concurrent access from worker threads or external systems will result in data corruption, race conditions, and undefined behavior. All modifications must be scheduled and executed within the server's primary update loop.

## API Surface
The public API consists of simple accessors and mutators for managing the component's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the MountPlugin registry. |
| getOriginalRoleIndex() | int | O(1) | Returns the NPC's original role index, stored so its behavior can be restored if it ceases to be a mount. |
| setOriginalRoleIndex(int) | void | O(1) | Sets the NPC's original role index. |
| getOwnerPlayerRef() | PlayerRef | O(1) | Returns a reference to the player entity currently owning or riding the mount. May be null. |
| setOwnerPlayerRef(PlayerRef) | void | O(1) | Assigns a player as the owner of this mount. |
| setAnchor(float, float, float) | void | O(1) | Sets the world coordinates of the mount's "home" or tethering point. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used internally by the engine for entity duplication. |

## Integration Patterns

### Standard Usage
The component should always be retrieved from an entity instance. Logic should then check and modify its state. The engine's systems will act upon these state changes in subsequent ticks.

```java
// A system retrieves an entity believed to be a mount
Entity mountEntity = world.getEntity(entityId);

// Retrieve the component to check its state
NPCMountComponent mount = mountEntity.getComponent(NPCMountComponent.class);

// Standard pattern: check for an owner before assigning a new one
if (mount != null && mount.getOwnerPlayerRef() == null) {
    mount.setOwnerPlayerRef(somePlayer.getRef());
    mount.setAnchor(mountEntity.getPosition()); // Set its home to its current location
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCMountComponent()` to modify an entity. This creates an orphaned component that is not tracked by any system. To add mount capabilities to an entity, use the `entity.addComponent()` method.
- **State Caching:** Avoid reading mount data into a local variable at the start of a complex function. The component's state can be altered by other systems or network events at any time. Always retrieve the component directly from the entity to ensure you are operating on the most recent data.
- **Cross-Thread Modification:** Never write to a component's fields from an asynchronous task or worker thread. This bypasses the engine's state management and will lead to severe concurrency issues.

## Data Pipeline
The NPCMountComponent is a node in the server's data flow, acting as both a target for state changes and a source for serialization.

> **Flow 1: Gameplay Interaction**
> Player Input -> Network Packet -> Server Command -> **MountSystem** -> *State change on* **NPCMountComponent**

> **Flow 2: World Persistence**
> Server Shutdown/Save Event -> EntityStore Serialization -> **NPCMountComponent**.CODEC -> *Serialized Data* -> Disk

> **Flow 3: Network Replication**
> Server Tick -> Entity Change Detection -> **NPCMountComponent**.CODEC -> *Serialized Data* -> Network Packet -> Client World


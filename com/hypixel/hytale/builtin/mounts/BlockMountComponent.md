---
description: Architectural reference for BlockMountComponent
---

# BlockMountComponent

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Component Data

## Definition
```java
// Signature
public class BlockMountComponent implements Component<ChunkStore> {
```

## Architecture & Concepts
The BlockMountComponent is a data-centric component within the server-side Entity-Component-System (ECS) framework. It serves as the authoritative state manager for static, world-based mountable objects, such as chairs, benches, or beds.

Unlike components that describe an entity's own properties (like Health or Velocity), this component represents a *relationship* between a block in the world and one or more entities. It is not attached to the player entity that is mounting; rather, it is attached to a logical representation of the mountable block itself, managed at the ChunkStore level.

Its primary responsibility is to track the occupancy of discrete "mount points" defined for a specific block type. For example, a bench block may define three separate mount points, and this component will track which entity, if any, occupies each one. It validates that the underlying block in the world has not changed since the component was created, ensuring data integrity.

## Lifecycle & Ownership
- **Creation:** A BlockMountComponent is instantiated on-demand by a higher-level system, typically the MountPlugin or a related interaction handler. This occurs the first time an entity attempts to mount a specific block that has mount points defined in its asset configuration. The constructor is populated with the block's current state (position, type, rotation) at that moment.

- **Scope:** The component's lifetime is tied directly to the occupancy state of the block it represents. It persists as long as at least one entity is mounted on the block. It is a server-side object with a scope limited to a single world.

- **Destruction:** The component is marked for garbage collection when its `isDead` method returns true, which occurs when the last seated entity is removed. The owning system, likely the ChunkStore's component manager, is responsible for its eventual removal. The component is also implicitly invalidated and should be destroyed if the world block at its `blockPos` is broken or its type or rotation changes.

## Internal State & Concurrency
- **State:** The component's state is highly mutable. Its core consists of two maps, `entitiesByMountPoint` and `mountPointByEntity`, which track the active entity-to-seat assignments. These maps are updated frequently as entities mount and dismount. It also holds an immutable snapshot of the block's expected state (`expectedBlockType`, `expectedRotation`) for validation purposes.

- **Thread Safety:** **This component is not thread-safe.** All methods that mutate its internal maps (`putSeatedEntity`, `removeSeatedEntity`, `clean`) are unsynchronized. It is designed to be accessed and modified exclusively from the main server game thread (the world tick thread). Off-thread access will result in race conditions, `ConcurrentModificationException`, and severe state corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| putSeatedEntity(mountPoint, seatedEntity) | void | O(1) | Assigns an entity to a specific mount point. This is the primary method for initiating a mount. |
| removeSeatedEntity(seatedEntity) | void | O(1) | Removes an entity from its mount point, freeing it up. |
| findAvailableSeat(targetBlock, choices, whereWasClicked) | BlockMountPoint | O(N) | Calculates the closest, unoccupied mount point from a list of candidates based on a world-space click location. Returns null if no seats are available. N is the number of choices. |
| isDead() | boolean | O(M) | Returns true if no entities are currently seated. This signals to the owning system that the component can be destroyed. M is the number of seated entities. |
| getSeatedEntities() | Collection | O(1) | Returns a collection of all entities currently mounted on this block. |

## Integration Patterns

### Standard Usage
A system handling player interactions would use this component to manage the "sitting" action. The logic involves finding the component for a target block, locating an available seat, and then updating the component's state.

```java
// System-level code (e.g., in a PlayerInteractionSystem)

// 1. Player interacts with a block at 'blockPos'
BlockMountComponent mount = findOrCreateMountComponentForBlock(blockPos);

// 2. Define the possible mount points for this block type
BlockMountPoint[] choices = getMountPointsForBlock(block);

// 3. Find the best seat based on where the player clicked
BlockMountPoint seat = mount.findAvailableSeat(blockPos, choices, player.getInteractionPoint());

// 4. If a seat is found, mount the entity
if (seat != null) {
    Ref<EntityStore> playerRef = player.getRef();
    mount.putSeatedEntity(seat, playerRef);
    
    // Further logic to update player position and animation state...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BlockMountComponent()` in general game logic. The component's lifecycle is managed by the ECS framework and the MountPlugin. Always retrieve it from or register it with the appropriate world system.

- **State Desynchronization:** The component only tracks the *association* between an entity and a mount point. Modifying the component by calling `putSeatedEntity` without also updating the entity's physical transform (position, rotation) and animation state will cause the server and client to become desynchronized. The entity will be logically "seated" but visually in the wrong place.

- **Leaking Components:** Failure to remove the component after the underlying block is destroyed will lead to orphaned components and memory leaks. Systems that handle block breaking must ensure they also clean up any associated BlockMountComponents.

## Data Pipeline
This component acts as a stateful controller rather than a data processor. The flow is driven by external events and results in state changes.

> Flow:
> Player Interaction Event -> Server Interaction System -> **BlockMountComponent.findAvailableSeat()** -> Server Interaction System -> **BlockMountComponent.putSeatedEntity()** -> Entity Transform System -> Network Packet to Client


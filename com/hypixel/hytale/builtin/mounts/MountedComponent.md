---
description: Architectural reference for MountedComponent
---

# MountedComponent

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Transient

## Definition
```java
// Signature
public class MountedComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The MountedComponent is a data-only component within Hytale's Entity-Component-System (ECS) architecture. It serves as a state flag, signifying that an entity's position, orientation, and lifecycle are currently subordinated to another object. This target object can be either another dynamic entity (e.g., a player riding a horse) or a static part of the world (e.g., a player sitting in a chair or operating a turret).

This component holds no logic. Instead, core engine systems query for its presence to alter their behavior. For example:
*   The **PhysicsSystem** will ignore standard physics calculations for an entity with a MountedComponent, instead calculating its transform based on the mount target's position and the component's attachment offset.
*   The **NetworkSyncSystem** uses this component's state to inform clients that an entity should be rendered as attached to another.
*   The **PlayerControlSystem** may delegate player input to the mount target if the MountController type dictates it.

It acts as a contract, decoupling the act of mounting from the various systems that need to react to it.

### Lifecycle & Ownership
- **Creation:** A MountedComponent is instantiated and attached to an EntityStore when a mount action is successfully executed. This is typically managed by a higher-level `MountSystem` in response to player interaction or AI behavior. It is never created in isolation.
- **Scope:** The component's lifetime is strictly bound to the duration of the mount. It is ephemeral and exists only while the entity is actively mounted.
- **Destruction:** The component is removed from its parent entity when a dismount event occurs. This action is also orchestrated by the `MountSystem`. Failure to remove this component will result in the entity being permanently and incorrectly attached to its former mount.

## Internal State & Concurrency
- **State:** This component is fundamentally **Mutable**. Its internal fields, particularly the reference to the mount target, define a temporary and volatile relationship between game objects. The `isNetworkOutdated` field is a mutable dirty flag, critical for efficient network replication.
- **Thread Safety:** **This component is not thread-safe.** As with most ECS components, it is designed for single-threaded access within the main server game loop (the world tick). Any and all modifications must be marshaled onto the main world thread. Unsynchronized access, especially to the `consumeNetworkOutdated` method, will cause severe race conditions and data corruption.

## API Surface
The public API is designed for read-only access by various systems, with one exception for network state consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the MountPlugin registry. |
| getMountedToEntity() | Ref<EntityStore> | O(1) | Returns a reference to the entity being mounted, or null if mounted to a block. |
| getMountedToBlock() | Ref<ChunkStore> | O(1) | Returns a reference to the world chunk containing the block being mounted, or null if mounted to an entity. |
| getControllerType() | MountController | O(1) | Returns an enum indicating which object controls movement (e.g., the rider or the mount). |
| consumeNetworkOutdated() | boolean | O(1) | **Warning:** State-mutating. Returns true if the component has changed since the last check, and atomically sets the internal flag to false. This is intended for exclusive use by the network synchronization system. |

## Integration Patterns

### Standard Usage
Systems should retrieve this component from an entity on a per-tick basis to determine if mount-specific logic should be applied. Do not hold references to this component across ticks.

```java
// Example from a hypothetical system that updates entity positions
void processEntity(EntityStore entity) {
    MountedComponent mountInfo = entity.getComponent(MountedComponent.class);

    if (mountInfo != null) {
        // This entity is mounted, apply special positioning logic
        Ref<EntityStore> mountTarget = mountInfo.getMountedToEntity();
        if (mountTarget != null && mountTarget.isAvailable()) {
            Vector3f offset = mountInfo.getAttachmentOffset();
            Vector3f newPosition = mountTarget.get().getPosition().add(offset);
            entity.setPosition(newPosition);
        }
    } else {
        // Apply standard physics and movement logic
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create this component with `new MountedComponent()`. Components must be added via `entity.addComponent()` to be correctly registered within the ECS world and its event systems.
- **State Caching:** Do not cache the component reference or the mount target reference across multiple game ticks. The mount relationship can be terminated at any moment, which would invalidate your cached reference and lead to crashes or undefined behavior.
- **External State Modification:** Other than the owning `MountSystem`, no system should modify the state of this component. It is a source of truth, not a scratchpad for calculations. Calling `consumeNetworkOutdated` from any system other than the `NetworkSyncSystem` will break entity replication.

## Data Pipeline
The MountedComponent serves as a data source that signals a change in an entity's behavior. Its state is authored by game logic and consumed by core systems.

> Flow:
> Player Interaction -> MountSystem -> **Creates/Attaches MountedComponent** -> PhysicsSystem (consumes transform data) -> NetworkSyncSystem (consumes `isNetworkOutdated` flag) -> Outgoing Network Packet


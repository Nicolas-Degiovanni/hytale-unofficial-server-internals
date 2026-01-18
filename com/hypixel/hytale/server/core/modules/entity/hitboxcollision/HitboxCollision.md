---
description: Architectural reference for HitboxCollision
---

# HitboxCollision

**Package:** com.hypixel.hytale.server.core.modules.entity.hitboxcollision
**Type:** Data Component

## Definition
```java
// Signature
public class HitboxCollision implements Component<EntityStore> {
```

## Architecture & Concepts
The HitboxCollision class is a data component within the server-side Entity Component System (ECS). It does not contain any logic. Its sole responsibility is to associate an Entity with a specific physical collision shape definition.

This component acts as a data-handle, storing an integer index (`hitboxCollisionConfigIndex`) that references a full `HitboxCollisionConfig` asset. This design pattern is an example of the Flyweight pattern; it avoids storing bulky configuration data on every entity, instead opting for a lightweight, shared reference. This is critical for performance in a server environment with potentially thousands of entities.

The component's state is managed by various server systems, primarily the physics and network synchronization systems. The `isNetworkOutdated` flag is a key part of the network replication pipeline, signaling to the network module that this component's state has changed and must be sent to clients.

## Lifecycle & Ownership
- **Creation:** An instance of HitboxCollision is created and attached to an Entity when that Entity is spawned into the world. This can occur in two primary ways:
    1. **Programmatically:** A game system creates a new Entity and attaches a new HitboxCollision component, typically by passing a `HitboxCollisionConfig` asset to its constructor.
    2. **Deserialization:** The static `CODEC` field is used by the world loading system to instantiate and populate the component from saved world data.
- **Scope:** The lifecycle of a HitboxCollision instance is strictly bound to the Entity it is attached to. It persists as long as the parent Entity exists in the `EntityStore`.
- **Destruction:** The component is marked for garbage collection when its parent Entity is removed from the world. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The component's state is mutable. It primarily consists of the `hitboxCollisionConfigIndex` and the `isNetworkOutdated` dirty flag. The state is considered "dirty" when it has been modified and has not yet been synchronized over the network.
- **Thread Safety:** This component is **not thread-safe**. Like most ECS components, it is designed to be accessed and modified exclusively by systems running on the main server thread. Unsynchronized access from other threads will lead to race conditions, particularly with the `consumeNetworkOutdated` method, resulting in corrupted network packets or missed updates. All modifications must be scheduled as tasks on the main game loop.

## API Surface
The public API is minimal, focusing on state access and interaction with the network synchronization loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the component's unique type identifier from the central `EntityModule` registry. |
| getHitboxCollisionConfigIndex() | int | O(1) | Returns the integer index of the associated `HitboxCollisionConfig` asset. |
| setHitboxCollisionConfigIndex(int) | void | O(1) | Directly sets the configuration index. **Warning:** This method does not automatically mark the component as network-outdated. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. Returns true if the component was outdated, false otherwise. This is the primary integration point for the network replication system. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used when duplicating entities. |

## Integration Patterns

### Standard Usage
This component is not meant to be used directly in most game logic. Instead, it is retrieved and processed by a dedicated physics or collision system, which iterates over entities possessing this component.

```java
// Example from within a server-side physics system
for (Entity entity : world.getEntitiesWith(HitboxCollision.class)) {
    HitboxCollision hitbox = entity.getComponent(HitboxCollision.class);
    int configIndex = hitbox.getHitboxCollisionConfigIndex();
    
    // Retrieve the full config from the asset map using the index
    HitboxCollisionConfig config = HitboxCollisionConfig.getAssetMap().get(configIndex);
    
    // ... perform collision detection logic using the config's shape data
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HitboxCollision()`. Components must be added to entities via the entity's own API (e.g., `entity.addComponent(...)`) to ensure they are correctly registered with the `EntityStore`.
- **State Caching:** Do not cache the value of `getHitboxCollisionConfigIndex()` across ticks. The component's state can be changed by other systems, and your system must always read the latest value.
- **External Flag Management:** Do not manually set the `isNetworkOutdated` flag. This is managed internally or by systems specifically designed for state synchronization. Calling `setHitboxCollisionConfigIndex` without a corresponding mechanism to flag the component for network updates will cause desynchronization between the server and clients.

## Data Pipeline
The data within this component follows distinct paths for persistence and network replication.

> **Persistence Flow (World Save/Load):**
> EntityStore -> Serializer -> **HitboxCollision.CODEC** -> World Data (Disk) -> Deserializer -> New **HitboxCollision** Instance

> **Network Replication Flow:**
> Game Logic changes state -> `isNetworkOutdated` flag is set to true -> Network System Tick -> Calls **consumeNetworkOutdated()** -> If true, serializes `hitboxCollisionConfigIndex` -> Network Packet -> Client


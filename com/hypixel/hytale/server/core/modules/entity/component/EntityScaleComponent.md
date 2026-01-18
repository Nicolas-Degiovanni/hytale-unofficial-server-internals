---
description: Architectural reference for EntityScaleComponent
---

# EntityScaleComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class EntityScaleComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The EntityScaleComponent is a fundamental data component within Hytale's server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to define the rendering scale, or size multiplier, of a game entity. As a component, it does not exist on its own; it is a piece of data that is attached to an Entity to grant it the "property" of having a scale.

This class acts as a simple data container, but it includes a critical mechanism for network synchronization: a dirty flag named isNetworkOutdated. This flag is essential for the engine's replication systems to efficiently determine when an entity's scale has changed and needs to be broadcast to clients.

The static CODEC field indicates that this component is serializable. This allows it to be persisted to disk as part of an Entity's data in the EntityStore and to be encoded for network transmission.

## Lifecycle & Ownership
- **Creation:** An EntityScaleComponent is created in one of two ways:
    1. **Programmatically:** By game logic, typically when an entity is first spawned or when its scale needs to be defined. For example: `entity.addComponent(new EntityScaleComponent(1.5f));`
    2. **Deserialization:** By the persistence or networking layers using the static CODEC. This occurs when loading an entity from the world save (EntityStore) or receiving entity data from a remote source.
- **Scope:** The lifecycle of an EntityScaleComponent is strictly bound to the parent Entity to which it is attached. It cannot exist independently.
- **Destruction:** The component is marked for garbage collection when its parent Entity is removed from the world via the EntityManager. There is no manual destruction method; its memory is managed entirely by the ECS framework.

## Internal State & Concurrency
- **State:** The component's state is mutable. It holds two pieces of data:
    - **scale:** A float representing the entity's size multiplier. This is the persistent state.
    - **isNetworkOutdated:** A boolean flag representing the transient network state. This flag is not saved to disk and exists only to manage runtime replication.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively from the main server game loop thread. Any mutation, such as calling setScale, from an asynchronous task or network thread will lead to race conditions and unpredictable behavior in the network synchronization system. All modifications must be scheduled to run on the main tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type identifier for this component from the EntityModule. |
| EntityScaleComponent(float) | constructor | O(1) | Creates a new instance with a specified initial scale. |
| getScale() | float | O(1) | Returns the current scale value. |
| setScale(float) | void | O(1) | Sets a new scale value and crucially marks the component as "dirty" for network synchronization. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. Returns true if the state was dirty. This is the primary integration point for the network replication engine. |
| clone() | Component | O(1) | Creates a new EntityScaleComponent instance with an identical scale value. The network dirty flag is not copied. |

## Integration Patterns

### Standard Usage
The component should always be accessed through an Entity instance. Game logic queries for the component, modifies it, and relies on the engine to handle the consequences (e.g., network updates).

```java
// Assume 'world' is the server world and 'entityId' is a valid entity
Entity entity = world.getEntityManager().getEntity(entityId);

// Check if the entity has the component before using it
if (entity.hasComponent(EntityScaleComponent.class)) {
    EntityScaleComponent scaleComp = entity.getComponent(EntityScaleComponent.class);

    // Modify the scale. This automatically flags it for network update.
    float currentScale = scaleComp.getScale();
    scaleComp.setScale(currentScale * 2.0f);
}
```

### Anti-Patterns (Do NOT do this)
- **Dangling Instances:** Do not create an instance of EntityScaleComponent and hold a reference to it without attaching it to an entity. It serves no purpose and will not be processed by any engine systems.
- **Manual Flag Management:** Never set the isNetworkOutdated flag directly. The setScale and consumeNetworkOutdated methods form a strict contract that must be respected for networking to function correctly.
- **Cross-Thread Modification:** Do not call setScale from a separate thread. This will corrupt the state of the network dirty flag and cause synchronization failures.

## Data Pipeline
The data within this component follows two primary pipelines: persistence and network replication.

> **Persistence Flow:**
> Game Logic modifies component -> World Save is triggered -> EntityManager iterates entities -> **EntityScaleComponent** is serialized via its CODEC -> Data written to EntityStore on disk.

> **Network Replication Flow:**
> Game Logic calls `setScale()` -> `isNetworkOutdated` flag is set to true -> Server Network System (end of tick) iterates dirty components -> Calls `consumeNetworkOutdated()` on **EntityScaleComponent** -> If true, component is serialized via its CODEC -> Data is added to a network packet and sent to clients.


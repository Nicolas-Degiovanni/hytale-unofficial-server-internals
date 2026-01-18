---
description: Architectural reference for BoundingBox
---

# BoundingBox

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Component / Data Object

## Definition
```java
// Signature
public class BoundingBox implements Component<EntityStore> {
```

## Architecture & Concepts
The BoundingBox is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. It does not contain any logic. Its sole purpose is to describe the physical volume of an entity for collision detection, physics simulation, and spatial queries.

This component acts as the primary data source for any system that needs to understand an entity's physical presence in the world. It separates the concept of an entity's physical shape from its other attributes, such as its position or velocity, which are managed by other components.

The design distinguishes between two levels of collision detail:
1.  **Primary BoundingBox:** A single, coarse `Box` object that encapsulates the entity's entire volume. This is used for broad-phase collision detection and performance-critical spatial lookups.
2.  **Detail Boxes:** An optional map of named, smaller `DetailBox` objects. These provide a more granular representation of the entity's shape, enabling narrow-phase collision checks for features like limb-specific damage or precise interactions. This data is typically defined in and loaded from an entity's model configuration asset.

## Lifecycle & Ownership
-   **Creation:** A BoundingBox is instantiated when an entity is created, typically by an `EntityFactory` processing an entity prefab. It is added to an `Entity` object as part of its initial component set. It is never created as a standalone object.
-   **Scope:** The lifecycle of a BoundingBox instance is strictly coupled to the `Entity` it is attached to. It persists as long as the parent entity exists within the game world.
-   **Destruction:** The component is marked for garbage collection when its parent `Entity` is destroyed and removed from the `EntityStore`. No manual de-allocation is required.

## Internal State & Concurrency
-   **State:** Highly mutable. The internal `Box` object is designed to be continuously updated by physics and movement systems, often on every server tick, to reflect the entity's current position and orientation in the world. The `detailBoxes` map, once set, is typically treated as immutable for the lifetime of the entity.
-   **Thread Safety:** **This component is not thread-safe.** It is designed for single-threaded access within the main server game loop. Modifying a BoundingBox from a worker thread or asynchronous task will lead to severe physics desynchronization, race conditions, and world state corruption. All interactions must be marshaled onto the main server thread.

## API Surface
The public API provides direct access to the component's state for high-performance systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for the BoundingBox component. |
| getBoundingBox() | Box | O(1) | Returns a direct reference to the mutable primary collision box. |
| setBoundingBox(Box) | void | O(1) | Copies the values from the provided Box into this component's internal Box. |
| getDetailBoxes() | Map | O(1) | Returns a reference to the map of named detail boxes. |
| setDetailBoxes(Map) | void | O(1) | Replaces the existing map of detail boxes. Primarily used during initialization. |
| clone() | Component | O(1) | Creates a new BoundingBox instance. **Warning:** This is a shallow copy. Only the primary `boundingBox` is cloned; the `detailBoxes` map is not copied to the new instance. |

## Integration Patterns

### Standard Usage
Systems, such as the PhysicsSystem, query the `EntityStore` for entities possessing a BoundingBox. They then read and update the box's state in conjunction with other components like Position and Velocity to simulate movement and collisions.

```java
// Example within a server-side System
for (Entity entity : world.getEntitiesWith(BoundingBox.class, Position.class)) {
    BoundingBox bbox = entity.getComponent(BoundingBox.getComponentType());
    Position pos = entity.getComponent(Position.getComponentType());

    // Synchronize the bounding box's center with the entity's world position
    // before running collision checks.
    bbox.getBoundingBox().setCenter(pos.getVector());
    
    // ... proceed with collision detection logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create a component using `new BoundingBox()`. Components must be added to an entity via the entity manager (e.g., `entity.addComponent()`) to ensure they are correctly registered within the ECS framework.
-   **Caching References:** Do not cache the `Box` object returned by `getBoundingBox()` across multiple ticks or outside the scope of a single system update. Its internal state is volatile and will be mutated by other systems. Always re-fetch the component from the entity to guarantee access to the most current state.
-   **Asynchronous Modification:** Never modify a BoundingBox from a separate thread. All physics and state mutations must occur on the main game thread to maintain world consistency.

## Data Pipeline
The BoundingBox component is a critical data point in the server's entity processing pipeline. It is primarily populated from static asset data and then becomes a dynamic, mutable state object within the game loop.

> Flow:
> Entity Prefab Asset -> EntityFactory -> **BoundingBox** (Component on Entity) -> PhysicsSystem (Reads/Writes) -> Collision Detection System (Reads)


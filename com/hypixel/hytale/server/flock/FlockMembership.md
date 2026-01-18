---
description: Architectural reference for FlockMembership
---

# FlockMembership

**Package:** com.hypixel.hytale.server.flock
**Type:** Transient

## Definition
```java
// Signature
public class FlockMembership implements Component<EntityStore> {
```

## Architecture & Concepts
The FlockMembership class is a data **Component** within Hytale's server-side Entity Component System (ECS). It does not contain logic; its sole purpose is to store state that defines an entity's relationship to a **Flock**. A Flock is a logical grouping of entities, managed by a higher-level system, to enable collective behaviors such as herding, swarming, or pack hunting.

This component acts as a data tag, allowing systems like the FlockSystem to efficiently query for all entities belonging to a specific group. It holds the unique identifier of the flock (flockId) and the entity's role within that group (membershipType), such as LEADER or MEMBER.

The inclusion of a Ref to EntityStore is a critical performance optimization. It serves as a direct, cached pointer to the flock's "leader" or "anchor" entity, avoiding expensive entity lookups by UUID during each game tick.

## Lifecycle & Ownership
- **Creation:** An instance of FlockMembership is created and attached to an entity under two primary conditions:
    1.  **Serialization:** When an entity is loaded from persistence (the EntityStore), the static CODEC is used to deserialize the entity's component data and construct a new FlockMembership instance.
    2.  **Runtime Logic:** A server system, such as a FlockManager or AI behavior tree, dynamically adds this component to an entity when it joins a flock.

- **Scope:** The lifecycle of a FlockMembership instance is strictly bound to the entity to which it is attached. It persists as long as the entity exists in the world and remains part of the flock.

- **Destruction:** The component is destroyed and subsequently garbage collected when its parent entity is removed from the world. The **unload** method is a critical part of this process, nullifying the internal flockRef to prevent memory leaks and break potential reference cycles before the object is garbage collected.

## Internal State & Concurrency
- **State:** The state of FlockMembership is **highly mutable**. Its fields, particularly membershipType and flockRef, are expected to change frequently at runtime as the flock's structure evolves (e.g., a new leader is chosen, or a member leaves the group). The flockRef field acts as a runtime cache for the flock's anchor entity.

- **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All interactions with a FlockMembership component must occur on the main server thread that processes the entity's game tick. Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. The ECS architecture relies on single-threaded access to components for performance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the ECS registry. |
| getFlockId() | UUID | O(1) | Returns the unique identifier of the flock this entity belongs to. |
| getFlockRef() | Ref<EntityStore> | O(1) | Returns a direct, possibly null, reference to the flock's anchor entity. |
| getMembershipType() | FlockMembership.Type | O(1) | Returns the entity's current role within the flock (e.g., LEADER, MEMBER). |
| unload() | void | O(1) | Clears the internal flockRef. Critical for cleanup to prevent memory leaks. |
| clone() | Component | O(N) | Creates a shallow copy of the component's data. Used by the engine for entity duplication. |

## Integration Patterns

### Standard Usage
FlockMembership is never used in isolation. It is always accessed via an entity reference by a managing system. The system queries for entities with this component to execute flocking logic.

```java
// A hypothetical FlockSystem processing an entity
void processEntity(Entity entity) {
    if (entity.hasComponent(FlockMembership.class)) {
        FlockMembership membership = entity.getComponent(FlockMembership.class);
        UUID flockId = membership.getFlockId();
        
        // ... use flockId and membershipType to apply flocking rules
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new FlockMembership()**. Components must be added to entities via the entity's own API, like **entity.addComponent(FlockMembership.class)**, to ensure they are correctly registered with the ECS.

- **State Management:** Do not use this component to store complex or persistent flock-wide data. It represents the membership status of a *single entity*. Global flock state should be managed by a dedicated system or a component on the flock's anchor entity.

- **Asynchronous Modification:** Never modify a FlockMembership component from a separate thread. All mutations must be queued and executed on the main server game loop to prevent catastrophic race conditions.

## Data Pipeline
FlockMembership is primarily a data container that participates in the server's serialization and game loop pipelines.

> **Serialization Flow:**
> Entity Save Event -> **FlockMembership** State -> Static CODEC -> Binary Representation -> EntityStore (Disk)

> **Game Loop Flow:**
> FlockSystem Tick -> Query for Entities with **FlockMembership** -> Read State -> AI Calculation -> Write to Transform/Other Components


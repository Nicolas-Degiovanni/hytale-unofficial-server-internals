---
description: Architectural reference for DisplayNameComponent
---

# DisplayNameComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class DisplayNameComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DisplayNameComponent is a fundamental data component within the server-side Entity Component System (ECS). Its sole responsibility is to associate a human-readable name, represented by a Message object, with an entity. It does not contain any logic; it is a pure data container managed by higher-level systems like the EntityModule.

A critical architectural feature is the static CODEC field. This BuilderCodec instance defines the serialization and deserialization contract for the component. The engine uses this codec to persist the component to disk within the EntityStore and to transmit its state over the network. This design decouples the component's data structure from the underlying storage or network format, allowing for versioning and flexible data representation.

This component is a leaf node in the entity object graph. Systems query for entities possessing this component to perform name-related operations, such as rendering nameplates, logging, or displaying chat messages.

## Lifecycle & Ownership
- **Creation:** A DisplayNameComponent is instantiated under two primary conditions:
    1. Programmatically, when a system adds the component to an entity via the EntityModule API.
    2. Automatically, by the persistence layer (EntityStore) or network layer when an entity is being deserialized from storage or a network packet. The static CODEC is invoked to construct the component from the source data.
- **Scope:** The lifecycle of a DisplayNameComponent is strictly bound to the lifecycle of its parent entity. It exists only as long as the entity it is attached to exists.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world. There is no explicit destruction method; memory is reclaimed by the JVM once all references from the EntityStore and parent entity are cleared.

## Internal State & Concurrency
- **State:** The component holds a single, mutable reference to a Message object, which can be null. The state is minimal and directly represents the entity's display name. It performs no caching or complex state management.
- **Thread Safety:** **This class is not thread-safe.** It is a simple Plain Old Java Object (POJO) with no internal synchronization mechanisms. All access and modification must be performed from the main server thread or be externally synchronized by the calling system (e.g., the EntityModule's update loop). Unmanaged multi-threaded access will lead to race conditions and undefined behavior.

## API Surface
The public API is minimal, focusing on data access and type registration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central EntityModule registry. |
| getDisplayName() | Message | O(1) | Returns the Message object representing the name, or null if not set. |
| clone() | Component | O(1) | Creates a new DisplayNameComponent instance with a reference to the same Message object. This is a shallow copy. |

## Integration Patterns

### Standard Usage
Systems should never hold a direct, long-lived reference to a component. Instead, they should query the parent entity for the component within the scope of a single operation or game tick.

```java
// Correctly retrieving the component from an entity
Entity entity = world.getEntity(entityId);
DisplayNameComponent nameComponent = entity.getComponent(DisplayNameComponent.getComponentType());

if (nameComponent != null) {
    Message displayName = nameComponent.getDisplayName();
    // Use the displayName for rendering, logging, etc.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DisplayNameComponent()`. Components must be added to entities via the appropriate entity management API, which ensures they are correctly registered with the EntityStore and all relevant systems.
- **State Caching:** Do not retrieve a component and store it in a local field for later use. The component can be removed from the entity at any time, which would lead to stale data and potential NullPointerExceptions. Always re-query the entity for the component when needed.
- **Cross-Thread Modification:** Never modify a component's state from a worker thread without explicit synchronization managed by the core engine. This will corrupt entity state.

## Data Pipeline
The DisplayNameComponent serves as a data point in several key engine pipelines.

> **World Load Pipeline:**
> Disk (EntityStore) -> `CODEC.decode()` -> **DisplayNameComponent** -> Attached to new Entity instance

> **Network Replication Pipeline:**
> Server Entity -> `CODEC.encode()` -> Network Packet -> Client -> `CODEC.decode()` -> **DisplayNameComponent** -> Attached to client-side Entity instance

> **Rendering Pipeline (Conceptual):**
> Game Tick -> Entity Query (for entities with DisplayNameComponent) -> **DisplayNameComponent.getDisplayName()** -> UI/Render System -> Nameplate Rendered in World


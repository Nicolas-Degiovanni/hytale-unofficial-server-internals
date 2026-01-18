---
description: Architectural reference for RotateObjectComponent
---

# RotateObjectComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class RotateObjectComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The RotateObjectComponent is a server-side, data-only component within Hytale's Entity-Component-System (ECS) architecture. Its sole responsibility is to store state describing the rotational velocity of an entity. It contains no game logic itself.

This component is designed to be attached to a server entity and subsequently processed by a dedicated *System*, such as a hypothetical RotationSystem. This System would query the world for all entities possessing a RotateObjectComponent and a TransformComponent, and on each server tick, apply the specified rotation to the entity's transform.

The static CODEC field is a critical feature, defining the serialization and deserialization contract for this component. This allows the component's state to be persisted to disk via the EntityStore and potentially synchronized over the network. The generic parameter `Component<EntityStore>` signifies its role as a piece of persistent world state managed by the server.

### Lifecycle & Ownership
- **Creation:** RotateObjectComponent instances are not typically instantiated directly by game logic. They are created by the entity framework during two main scenarios:
    1. **Deserialization:** When an entity is loaded from the EntityStore (world storage), the component's data is read and the CODEC is used to construct a new instance.
    2. **Programmatic Addition:** A high-level system or game script may add the component to an existing entity at runtime.
- **Scope:** The lifecycle of a RotateObjectComponent is strictly bound to the entity it is attached to. It exists only as long as its parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single float value, rotationSpeed. It is intended to be frequently read by systems and potentially modified by game logic (e.g., a spell that temporarily changes an object's rotation speed).
- **Thread Safety:** **This class is not thread-safe.** Like most ECS components, it is designed for single-threaded access within the main server game loop. Modifying or reading component data from asynchronous tasks or network threads without proper synchronization will lead to race conditions, world state corruption, and server instability. All interactions must be marshaled to the main server thread.

## API Surface
The public API is minimal, focusing exclusively on state management and type identification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component class from the EntityModule. |
| setRotationSpeed(float) | void | O(1) | Sets the rotational velocity. |
| getRotationSpeed() | float | O(1) | Retrieves the current rotational velocity. |

## Integration Patterns

### Standard Usage
This component is not used in isolation. It is retrieved from an entity instance within a System that is responsible for processing it.

```java
// Example from within a hypothetical server-side RotationSystem
void processEntity(Entity entity, float deltaTime) {
    // Retrieve the component using its static type definition
    RotateObjectComponent rotation = entity.getComponent(RotateObjectComponent.getComponentType());
    TransformComponent transform = entity.getComponent(TransformComponent.getComponentType());

    if (rotation != null && transform != null) {
        float speed = rotation.getRotationSpeed();
        // Apply rotation logic to the entity's transform
        transform.rotate(Vector3.UP, speed * deltaTime);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Attachment:** While you can use `new RotateObjectComponent()`, it is not the standard way to add it to an entity. The entity's own API should be used to ensure it is correctly registered.
- **Logic in Component:** Do not add methods containing game logic to this class. All logic that acts upon this component's data should reside in a separate System.
- **Asynchronous Modification:** Never modify the rotationSpeed from a separate thread. All state changes must be queued for execution on the main server thread.

## Data Pipeline
The RotateObjectComponent acts as a simple data container. Its data flows from storage or game events, is processed by game systems, and its *effects* are ultimately observed by the client.

> **Flow (World Load):**
> Disk Storage (EntityStore) -> Deserializer (using CODEC) -> **RotateObjectComponent** instance -> Attached to Entity

> **Flow (Runtime Logic):**
> Game System (e.g., RotationSystem) -> Reads **RotateObjectComponent**.rotationSpeed -> Updates Entity's TransformComponent -> Network Marshaller -> Client receives updated transform -> Visual rotation occurs

---


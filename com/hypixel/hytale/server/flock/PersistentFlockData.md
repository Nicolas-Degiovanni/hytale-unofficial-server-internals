---
description: Architectural reference for PersistentFlockData
---

# PersistentFlockData

**Package:** com.hypixel.hytale.server.flock
**Type:** Data Component

## Definition
```java
// Signature
public class PersistentFlockData implements Component<EntityStore> {
```

## Architecture & Concepts

PersistentFlockData is a server-side **Data Component** within Hytale's Entity Component System (ECS). Its sole responsibility is to encapsulate the state of a flock that must persist across server restarts and be saved with the world. It acts as a data container for core flock properties, such as size constraints and member roles.

This component does not contain any behavioral logic. Instead, it serves as the "source of truth" for other, more complex systems, like the Flock AI or FlockManager, which read from and write to this component during the game loop.

A critical feature is the static **CODEC** field. This integrates the component directly with the engine's serialization framework, enabling the EntityStore to automatically manage saving and loading its state to and from disk. This design decouples the flock's data representation from the systems that operate on it.

## Lifecycle & Ownership

-   **Creation:** An instance of PersistentFlockData is never created directly by a developer. It is instantiated under two specific conditions:
    1.  **Programmatically:** By a higher-level system, such as a FlockManager, when a new flock is spawned in the world. This typically involves passing a FlockAsset definition to configure its initial state.
    2.  **Deserialization:** By the EntityStore during the world loading process. The engine uses the component's static CODEC to construct the object from its serialized representation on disk.

-   **Scope:** The lifecycle of a PersistentFlockData instance is strictly bound to the entity to which it is attached. It persists as long as the parent entity exists within the server's world state.

-   **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world. There is no manual destruction method; the ECS and the Java garbage collector manage its cleanup.

## Internal State & Concurrency

-   **State:** The component's state is **mutable**. Fields like *size* are expected to change frequently as entities join or leave the flock. The *maxGrowSize* and *flockAllowedRoles* fields are typically configured at creation and treated as read-only for the remainder of the component's lifetime.

-   **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed and modified exclusively by the main server thread responsible for the entity's game logic. Any concurrent modification from other threads will result in race conditions and data corruption. All interactions must be synchronized with the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier used to register this component with the ECS. |
| isFlockAllowedRole(String role) | boolean | O(log N) | Checks if a given role is permitted in the flock. Utilizes an efficient binary search on a pre-sorted internal array. |
| increaseSize() | void | O(1) | Increments the current size of the flock. This is not an atomic operation. |
| decreaseSize() | void | O(1) | Decrements the current size of the flock. This is not an atomic operation. |
| clone() | Component | O(1) | Creates a shallow copy of the component. **Warning:** The internal array of allowed roles is shared, not deep-copied. |

## Integration Patterns

### Standard Usage

This component should always be retrieved from an existing entity. Game logic then reads its state to make decisions or modifies its state to reflect changes in the world.

```java
// Assume 'flockEntity' is an entity with this component attached
PersistentFlockData flockData = flockEntity.getComponent(PersistentFlockData.getComponentType());

if (flockData != null && flockData.isFlockAllowedRole("guardian")) {
    // Logic for guardian roles...
    flockData.increaseSize();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new PersistentFlockData()`. The private constructor exists for the serialization system. Creating it manually bypasses the ECS attachment and persistence mechanisms, resulting in an orphaned object that will not be saved.

-   **Cross-Thread Modification:** Do not access an instance of this component from a worker thread. All reads and writes must be performed on the main server thread to prevent race conditions.

-   **Post-Creation State Mutation:** Do not modify the array returned by `flockAllowedRoles` after the component has been created. The `isFlockAllowedRole` method relies on this array being sorted; external modifications will break this contract and lead to undefined behavior.

## Data Pipeline

PersistentFlockData primarily participates in the world persistence pipeline, not a real-time data processing pipeline.

> **Save Flow:**
> Game Logic -> Modifies **PersistentFlockData** on an Entity -> World Save Event -> EntityStore -> **PersistentFlockData.CODEC** (Serialize) -> Binary Data on Disk

> **Load Flow:**
> World Load Event -> EntityStore -> Reads Binary Data from Disk -> **PersistentFlockData.CODEC** (Deserialize) -> New **PersistentFlockData** instance attached to Entity


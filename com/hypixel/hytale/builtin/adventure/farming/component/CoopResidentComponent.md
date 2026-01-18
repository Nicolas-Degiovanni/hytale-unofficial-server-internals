---
description: Architectural reference for CoopResidentComponent
---

# CoopResidentComponent

**Package:** com.hypixel.hytale.builtin.adventure.farming.component
**Type:** Data Component

## Definition
```java
// Signature
public class CoopResidentComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The CoopResidentComponent is a data-only component within Hytale's Entity Component System (ECS) framework. It serves as a data-bag, attaching state information to an entity, typically a farm animal, that links it to a specific coop structure in the game world.

This component is fundamental to the **Farming System**. It enables systems like animal AI, population management, and world persistence to reason about an entity's "home" or designated habitat. Its primary responsibilities are:

1.  **Spatial Association:** Storing the world coordinate of the entity's assigned coop via the coopLocation field. This allows AI systems to guide the entity back to its home.
2.  **Lifecycle Management:** Providing a flag, markedForDespawn, which signals to other systems that the entity is a candidate for removal from the world. This is a critical part of managing entity populations and performance, preventing an unbounded number of animals from accumulating.

The component's serialization is handled by a static CODEC field, which allows the engine to save and load an entity's coop association with the world data.

## Lifecycle & Ownership
The lifecycle of a CoopResidentComponent is strictly bound to the entity to which it is attached. It does not and cannot exist independently.

-   **Creation:** A CoopResidentComponent is instantiated under two conditions:
    1.  **Programmatically:** By a high-level game system, such as a FarmingSystem, when an entity is spawned or assigned to a coop.
    2.  **Deserialization:** By the world loader, which uses the component's CODEC to reconstruct it from saved chunk data when an entity is loaded into the simulation.
-   **Scope:** Its lifetime is identical to that of its parent entity. It persists as long as the entity exists in the world and retains this component.
-   **Destruction:** The component is destroyed and garbage collected when its parent entity is removed from the world or when a system explicitly removes the component from the entity.

**WARNING:** Do not hold long-lived references to this component. Always re-acquire it from the entity each game tick, as it can be removed at any time by another system.

## Internal State & Concurrency
-   **State:** The component's state is entirely **mutable**. Both the coopLocation and markedForDespawn fields are designed to be modified at runtime by game logic. The internal Vector3i object is also mutable. The provided clone method performs a deep-enough copy to ensure value semantics when duplicating the component.
-   **Thread Safety:** This component is **not thread-safe**. Like most ECS components, it is designed to be accessed and modified exclusively by its owning systems within a single, well-defined phase of the main game loop. Unsynchronized access from other threads will lead to race conditions, data corruption, and unpredictable behavior. All modifications must be marshaled back to the main game thread.

## API Surface
The public API consists of simple accessors and a static type resolver. Business logic is intentionally absent, as that is the responsibility of a System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component class. |
| setCoopLocation(Vector3i) | void | O(1) | Sets the world location of the associated coop. |
| getCoopLocation() | Vector3i | O(1) | Returns the world location of the associated coop. |
| setMarkedForDespawn(boolean) | void | O(1) | Flags the entity for future despawning. |
| getMarkedForDespawn() | boolean | O(1) | Checks if the entity is flagged for despawning. |
| clone() | Component | O(1) | Creates a new instance of the component with copied state. |

## Integration Patterns

### Standard Usage
A system (e.g., a DespawningSystem or AnimalAISystem) retrieves the component from an entity to read or write its state. Direct interaction with the constructor is forbidden.

```java
// Example within a hypothetical System's update method
void processEntity(Entity entity) {
    CoopResidentComponent resident = entity.getComponent(CoopResidentComponent.getComponentType());

    if (resident != null) {
        if (isTooFarFromCoop(entity.getPosition(), resident.getCoopLocation())) {
            resident.setMarkedForDespawn(true);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CoopResidentComponent()`. Components must be added to entities via the world or entity APIs, which ensures they are correctly registered with the ECS framework.
-   **Cross-Thread Modification:** Do not read or write to this component from an asynchronous task or worker thread without explicit synchronization managed by the calling system. The component itself provides no guarantees.
-   **State Caching:** Do not retrieve a component and store it in a field for later use. The component may be removed from the entity, leading to a stale reference. Fetch it from the entity every time you need to interact with it.

## Data Pipeline
The CoopResidentComponent acts as a data container within the larger ECS data flow. Its primary role is to persist entity state between game sessions and to communicate state between different game systems during a single frame.

> **Persistence Flow:**
> World Save (Disk) -> World Loader -> **CODEC Deserializer** -> **CoopResidentComponent** (In Memory) -> Game Session -> **CODEC Serializer** -> World Save (Disk)

> **Game Loop Flow:**
> AnimalAISystem (Reads coopLocation) -> **CoopResidentComponent** <- DespawnSystem (Writes markedForDespawn)


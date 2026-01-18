---
description: Architectural reference for FromWorldGen
---

# FromWorldGen

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class FromWorldGen implements Component<EntityStore> {
```

## Architecture & Concepts
The FromWorldGen component is a simple, server-side data container within the Entity-Component-System (ECS) architecture. Its sole purpose is to act as a "marker" or "tag" that links a game entity to its origin within the procedural world generation system.

When the server generates a new world chunk, it may place entities such as creatures, chests, or spawners. By attaching a FromWorldGen component to these entities, the engine permanently records their provenance. The integer field, worldGenId, serves as a unique identifier referencing the specific rule, spawner, or procedural event that created the entity.

This component is critical for systems that need to differentiate between entities spawned organically during gameplay (e.g., by a player or a dynamic spawner) and those placed deterministically at world creation. For example, a cleanup or respawn system might query for entities with this component to decide whether to remove or regenerate them when a chunk is reloaded.

## Lifecycle & Ownership
- **Creation:** An instance of FromWorldGen is created exclusively by the server's world generation pipeline. When a procedural rule dictates the placement of an entity, the generator instantiates this component and attaches it to the new entity before it is added to the world's EntityStore.
- **Scope:** The lifecycle of a FromWorldGen component is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection and destroyed when its parent entity is removed from the game world. There are no manual destruction methods; its lifetime is managed entirely by the parent entity.

## Internal State & Concurrency
- **State:** The component holds a single piece of mutable state: the private integer worldGenId. However, this value should be considered **immutable after construction**. Modifying it post-creation would corrupt the entity's link to its origin and is a severe anti-pattern. The state is serialized to and from persistent storage via the static CODEC field.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. In the server's ECS architecture, all component access and modification for a given entity must be synchronized by the owning system or performed exclusively on the main game loop thread. Unmanaged multi-threaded access will lead to race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FromWorldGen(int) | constructor | O(1) | Creates a new component instance. Intended for internal use by the world generator. |
| getWorldGenId() | int | O(1) | Returns the unique identifier linking the entity to its world generation source. |
| getComponentType() | static ComponentType | O(1) | Retrieves the component's unique type definition from the central EntityModule registry. |
| clone() | Component | O(1) | Creates a deep copy of the component. Used for entity duplication or snapshotting. |

## Integration Patterns

### Standard Usage
Systems should query for entities possessing this component to perform world-generation-specific logic. Direct construction is not a standard use case for gameplay systems.

```java
// Example of a system checking if an entity originated from world generation
Entity entity = ...;
Optional<FromWorldGen> worldGenInfo = entity.getComponent(FromWorldGen.getComponentType());

if (worldGenInfo.isPresent()) {
    int id = worldGenInfo.get().getWorldGenId();
    // Logic for entities placed by the world generator
    System.out.println("Entity " + entity.getId() + " was created by WorldGen rule " + id);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Attachment:** Do not manually create and attach a FromWorldGen component to an entity after world generation. This falsifies the entity's origin and will cause unpredictable behavior in persistence and respawn systems.
- **State Mutation:** Never modify the worldGenId after the component has been created. The ID is a permanent record of origin.
- **Client-Side Logic:** This is a server-authoritative component. Client-side code should never assume its presence or attempt to interact with it.

## Data Pipeline
The FromWorldGen component is primarily a data source for other systems and a payload for the persistence engine. It does not process data itself.

> **Creation Flow:**
> World Generation Rule -> Instantiate Entity -> **Instantiate FromWorldGen(id)** -> Attach Component to Entity -> Add Entity to EntityStore

> **Persistence Flow (World Save):**
> EntityStore Save Trigger -> Entity Serialization -> **FromWorldGen.CODEC serializes component to JSON/binary** -> Write to Disk

> **Loading Flow (World Load):**
> Read from Disk -> Entity Deserialization -> **FromWorldGen.CODEC creates component from data** -> Attach Component to re-created Entity -> Add Entity to EntityStore


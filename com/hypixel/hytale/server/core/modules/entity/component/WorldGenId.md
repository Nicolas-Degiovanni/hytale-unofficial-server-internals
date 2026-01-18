---
description: Architectural reference for WorldGenId
---

# WorldGenId

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class WorldGenId implements Component<EntityStore> {
```

## Architecture & Concepts
The WorldGenId component is a fundamental data container within the server-side Entity-Component-System (ECS) architecture. It does not contain any logic; its sole purpose is to attach a persistent identifier to an entity, linking it directly to the world generation process that created it.

This component acts as a permanent "birth certificate" for an entity. It allows other systems to distinguish between entities spawned dynamically during gameplay and those placed procedurally during chunk generation. This distinction is critical for systems managing respawning, loot tables, or environmental consistency. For example, a specific, named creature spawned by world generation should be respawned upon chunk reload, whereas a randomly spawned wolf should not.

The static CODEC field is the primary integration point with Hytale's data serialization framework. It defines a non-negotiable contract for how this component's state is written to and read from the EntityStore, ensuring data integrity across server sessions.

## Lifecycle & Ownership
- **Creation:** WorldGenId instances are not intended for direct instantiation by game logic systems. They are created under two specific conditions:
    1. By a world generation pipeline when a new chunk is populated and a procedural entity is placed in the world.
    2. By the serialization framework via its CODEC when an entity is loaded from the EntityStore on disk.
- **Scope:** The lifecycle of a WorldGenId component is strictly and exclusively bound to the lifecycle of its parent entity. It cannot exist independently.
- **Destruction:** The component is marked for garbage collection implicitly when its parent entity is destroyed. There are no manual cleanup or `destroy` methods.

## Internal State & Concurrency
- **State:** The component's state is effectively immutable. It holds a single integer, `worldGenId`, which is set during construction and cannot be modified thereafter. The absence of setters enforces this design. The `clone` method produces a new, distinct instance rather than modifying the existing one.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, the entity to which it is attached is not guaranteed to be thread-safe. All access to an entity's components, including WorldGenId, must be synchronized through the main server thread or the entity's designated processing system. Direct, multi-threaded access to an entity's component map will lead to severe concurrency violations.

## API Surface
The public API is minimal, reflecting its role as a simple data holder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the EntityModule registry. |
| getWorldGenId() | int | O(1) | Returns the unique world generation identifier. |
| clone() | Component | O(1) | Creates a new WorldGenId instance with an identical internal ID. |

## Integration Patterns

### Standard Usage
The correct pattern for using this component is to retrieve it from an existing entity to query its origin. It is a read-only operation from the perspective of game logic.

```java
// How a developer should normally use this
// Assume 'serverEntity' is a valid entity instance
serverEntity.getComponent(WorldGenId.getComponentType()).ifPresent(worldGenIdComponent -> {
    int id = worldGenIdComponent.getWorldGenId();

    if (id != WorldGenId.NON_WORLD_GEN_ID) {
        // This entity was placed by the world generator.
        // Apply special persistence or respawn logic here.
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldGenId()` in standard game systems. Components must be managed by the `EntityManager` or an equivalent entity factory to ensure they are correctly registered and attached to an entity. Manual creation bypasses the ECS framework entirely.
- **State Mutation:** Do not attempt to modify the `worldGenId` field after construction, for example, through Java Reflection. This identifier is considered permanent and is used as a key for persistent storage. Altering it will corrupt world state and lead to unpredictable behavior on server restart.

## Data Pipeline
WorldGenId is a passive participant in the server's data persistence pipeline. It is a payload that is serialized and deserialized by higher-level systems.

> **Serialization Flow (Server Shutdown / Chunk Unload):**
> Entity Unload Event -> EntityManager -> **WorldGenId** -> WorldGenId.CODEC -> EntityStore -> Disk

> **Deserialization Flow (Server Startup / Chunk Load):**
> Disk -> EntityStore -> WorldGenId.CODEC -> **WorldGenId Instance** -> Attached to new Entity by EntityManager


---
description: Architectural reference for LocalSpawnBeacon
---

# LocalSpawnBeacon

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** Transient

## Definition
```java
// Signature
public class LocalSpawnBeacon implements Component<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnBeacon is a server-side **marker component** within Hytale's Entity Component System (ECS). Its sole architectural purpose is to flag an entity, designating it as a valid spawn location for other entities. It carries no data; its presence on an entity is the signal.

This component is a fundamental building block for the server's spawning logic, managed by the SpawningPlugin. Systems responsible for player respawns or NPC population will query the world's EntityStore for entities that possess a LocalSpawnBeacon component. The position of the parent entity is then used as a candidate spawn point.

By implementing Component<EntityStore>, it explicitly declares its lifecycle is managed within the server's primary world entity container, not on the client.

### Lifecycle & Ownership
- **Creation:** An instance is created and attached to an entity when game logic dictates a new spawn point should exist. This can occur when a player places a bed, an administrator sets a world spawn, or a structure with a predefined spawn point is generated. It is also instantiated by the persistence layer via its CODEC when loading world data.
- **Scope:** The lifecycle of a LocalSpawnBeacon instance is strictly bound to the entity it is attached to. It exists only as long as its parent entity exists.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the EntityStore. There is no manual destruction method.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no member fields. Its design as a marker component means its value is binary—it either exists on an entity or it does not. This simplicity is critical for performance in large-scale ECS queries.
- **Thread Safety:** Inherently **thread-safe**. As an immutable, stateless object, a LocalSpawnBeacon instance can be safely read from any thread.

    **WARNING:** While the component itself is thread-safe, the parent entity and the EntityStore it belongs to are not. All modifications to an entity, including adding or removing this component, must be performed on the main server thread or follow the engine's specific concurrency protocols for world modification.

## API Surface
The public API is minimal, reflecting its role as a data-less component and adhering to the contracts of the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, registered type identifier for this component from the SpawningPlugin. |
| clone() | Component | O(1) | Creates a new instance of LocalSpawnBeacon. Required by the Component interface for entity duplication. |

## Integration Patterns

### Standard Usage
This component is not meant to be interacted with directly. Instead, game systems add it to entities to mark them, and other systems query for its presence.

**Attaching the component:**
```java
// Example: A player places a bed, which becomes a spawn point
Entity bedEntity = createBed();
bedEntity.addComponent(new LocalSpawnBeacon());
world.addEntity(bedEntity);
```

**Querying for the component:**
```java
// Example: A spawning system finds all available spawn points
List<Entity> spawnPoints = world.query()
                                .with(LocalSpawnBeacon.getComponentType())
                                .execute();
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not modify this class to include data, such as a spawn radius or player owner. If such data is needed, create a separate component (e.g., SpawnPointData) and attach it to the same entity. The performance of marker component queries relies on this class remaining stateless.
- **Manual Instantiation for Checks:** Avoid creating a new instance just to perform a type check. Use the framework's intended methods.

    **INCORRECT:**
    `if (entity.getComponent(LocalSpawnBeacon.class) != null) { ... }`

    **CORRECT:**
    `if (entity.hasComponent(LocalSpawnBeacon.getComponentType())) { ... }`

## Data Pipeline
The LocalSpawnBeacon does not process data. Instead, its existence *is* the data. The pipeline describes how its presence is used by the spawning system.

> Flow:
> Spawning System Timer -> ECS Query for **LocalSpawnBeacon** -> Filter valid entities -> Extract Position Component from entity -> World Spawner -> Create New Entity at Position


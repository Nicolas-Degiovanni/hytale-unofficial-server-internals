---
description: Architectural reference for AmbientEmitterComponent
---

# AmbientEmitterComponent

**Package:** com.hypixel.hytale.builtin.ambience.components
**Type:** Data Component

## Definition
```java
// Signature
public class AmbientEmitterComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The AmbientEmitterComponent is a data-only component within Hytale's Entity Component System (ECS). It does not contain any logic itself; instead, it serves as a data blueprint that attaches ambient sound behavior to an entity.

Its primary role is to flag an entity as a *source* or *spawner* for a persistent ambient sound. An associated system, likely part of the AmbiencePlugin, queries for entities possessing this component. Upon finding one, the system reads the configured **soundEventId** and is responsible for instantiating a separate, dedicated sound-emitting entity in the world.

This component acts as the declarative link between a logical position in the world (the entity it's attached to) and a specific sound effect defined elsewhere in the game's assets. The **spawnedEmitter** field is a runtime-only cache, holding a reference to the live entity that was created to fulfill the sound request. This decouples the spawner's lifecycle from the sound emitter's lifecycle.

### Lifecycle & Ownership
- **Creation:** Instances are created through two primary mechanisms:
    1.  **Deserialization:** The engine's Codec system instantiates the component when loading an entity from a world save or a prefab definition file. The static CODEC field defines how to map the data file's *SoundEventId* key to the internal field.
    2.  **Cloning:** The component is duplicated via its clone method when an entity template is instantiated in the game world. Note that the clone operation is shallow; it only copies the configuration (soundEventId) and explicitly discards any runtime state like the spawnedEmitter reference.

- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the EntityStore.

- **Destruction:** The component is marked for garbage collection and its memory is reclaimed when its parent entity is destroyed or the component is explicitly removed from the entity.

## Internal State & Concurrency
- **State:** The component's state is **mutable**. The soundEventId can be changed post-creation, and the spawnedEmitter reference is designed to be written to at runtime by a managing system. The state is divided into two categories:
    - **Configuration State:** The soundEventId.
    - **Runtime State:** The spawnedEmitter reference, which is transient and not serialized.

- **Thread Safety:** This component is **not thread-safe** and must be considered thread-hostile. As with most ECS components, all reads and writes must be performed on the main world thread to prevent race conditions, memory visibility issues, and inconsistent state. Unsynchronized access from other threads is an error.

## API Surface
The public API is minimal, consisting primarily of data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type for this component from the AmbiencePlugin. |
| getSoundEventId() | String | O(1) | Returns the identifier for the sound event this component is configured to emit. |
| setSoundEventId(String) | void | O(1) | Sets the sound event identifier. **Warning:** Modifying this at runtime may not have an effect unless a managing system is polling for changes. |
| getSpawnedEmitter() | Ref | O(1) | Retrieves the runtime reference to the live entity currently emitting the sound. May be null. |
| setSpawnedEmitter(Ref) | void | O(1) | Sets the runtime reference. **Warning:** This should only be called by the owning Ambience system. |
| clone() | Component | O(1) | Creates a shallow copy of the component, preserving configuration but not runtime state. |

## Integration Patterns

### Standard Usage
This component is designed to be defined within entity prefabs. A managing system handles all runtime logic. A developer should never need to interact with this component directly in game logic code.

```java
// This is a conceptual example of how a system would use the component.
// Do not replicate this logic; it is handled by the engine.

for (Entity entity : world.query(AmbientEmitterComponent.class)) {
    AmbientEmitterComponent emitter = entity.get(AmbientEmitterComponent.class);

    // Check if a sound emitter has already been spawned for this source
    if (emitter.getSpawnedEmitter() == null) {
        String soundId = emitter.getSoundEventId();
        Entity soundEntity = spawnSoundEntityAt(entity.getPosition(), soundId);
        
        // Cache the reference to the live entity
        emitter.setSpawnedEmitter(soundEntity.getRef());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AmbientEmitterComponent()`. Components must be added to entities via the appropriate EntityManager or world APIs to ensure they are correctly registered with the ECS.
- **Manual State Management:** Do not call `setSpawnedEmitter` from general game logic. This field is exclusively owned and managed by the core Ambience system to prevent reference leaks or state corruption.
- **Cross-Thread Access:** Never read or write to this component from an asynchronous task or a different thread without explicit synchronization with the main world thread. This will lead to unpredictable behavior.

## Data Pipeline
The AmbientEmitterComponent primarily serves as an entry point for the ambience data flow. It translates static world data into a runtime request for the audio system.

> Flow:
> Prefab/World Data (JSON) -> Engine Deserializer (CODEC) -> **AmbientEmitterComponent** (on spawner entity) -> Ambience System (reads component) -> Entity Spawn Request -> Audio Engine (plays sound from new entity)


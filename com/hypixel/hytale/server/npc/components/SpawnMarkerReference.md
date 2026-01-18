---
description: Architectural reference for SpawnMarkerReference
---

# SpawnMarkerReference

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Component (Data-Oriented)

## Definition
```java
// Signature
public class SpawnMarkerReference extends SpawnReference implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnMarkerReference is a server-side, data-only component within the Entity-Component-System (ECS) architecture. Its sole purpose is to create a persistent link between a spawned entity (typically an NPC) and the specific *spawn marker* entity that created it.

This component acts as a data tag, holding a reference to the original spawner. This relationship is crucial for game logic systems, such as:
- **Respawning Logic:** A `RespawnSystem` can query for entities with this component to determine where they should be recreated after being destroyed.
- **Zone Behavior:** AI systems can use this reference to determine an NPC's "home" location, preventing it from wandering too far from its spawn point.
- **Spawner Management:** A system could use this to track how many active entities a specific spawner is responsible for.

The component's type is registered with the ECS framework via the `SpawningPlugin`, making it a discoverable and manageable part of the server's spawning module. Its existence on an entity is a definitive statement that the entity's lifecycle is managed by a spawner.

## Lifecycle & Ownership
- **Creation:** A SpawnMarkerReference is instantiated and attached to an entity by a spawning system at the moment of the entity's creation. It is also created during world loading when the `CODEC` deserializes the component's data from the `EntityStore`.
- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the entity exists in the world.
- **Destruction:** The component is destroyed automatically by the `EntityStore` when its parent entity is removed from the world. Manual removal is rare and should be handled by a dedicated game system.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single `reference` field inherited from its parent, SpawnReference. This field holds an identifier pointing to the spawn marker entity. The state is designed to be persisted via its associated `CODEC`.
- **Thread Safety:** This component is **not thread-safe**. Like most ECS components, it is designed to be accessed and modified exclusively by systems operating on the server's main game loop thread. Unsynchronized access from other threads will lead to race conditions and world state corruption.

## API Surface
The public contract is minimal, focusing on ECS integration rather than direct manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the SpawningPlugin. Essential for ECS lookups. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used by the ECS framework when duplicating entities. |

## Integration Patterns

### Standard Usage
Systems should retrieve this component from an entity to query its spawn point. Direct modification of the internal reference is discouraged; this is typically set once at creation.

```java
// A hypothetical system checking an NPC's distance from its spawn point
Entity npc = ...;
SpawnMarkerReference spawnRef = npc.getComponent(SpawnMarkerReference.getComponentType());

if (spawnRef != null) {
    EntityId markerId = spawnRef.reference;
    // Use markerId to find the spawn marker entity and its position
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SpawnMarkerReference()`. Components must be added to entities via the `EntityStore` or a world management API to ensure they are correctly registered and tracked by the ECS.
- **Manual Reference Management:** Do not manually change the `reference` field after an entity has been spawned. This can break respawn logic and lead to orphaned entities. This field should be treated as immutable after initial creation.

## Data Pipeline
This component primarily represents data-at-rest but is a key element in the entity persistence and spawning data flows.

> **Persistence (Save) Flow:**
> World Save Trigger -> EntityStore Serialization -> **SpawnMarkerReference.CODEC** -> Serialized Entity Data -> Disk

> **Persistence (Load) Flow:**
> Disk -> Serialized Entity Data -> EntityStore Deserialization -> **SpawnMarkerReference.CODEC** -> Live Component on Entity

> **Spawning Logic Flow:**
> SpawningSystem locates a Spawn Marker -> System creates a new NPC Entity -> System adds **SpawnMarkerReference** to NPC (pointing to marker) -> NPC is added to the world


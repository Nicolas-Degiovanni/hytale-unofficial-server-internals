---
description: Architectural reference for SpawnBeaconReference
---

# SpawnBeaconReference

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Transient Data Component

## Definition
```java
// Signature
public class SpawnBeaconReference extends SpawnReference implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnBeaconReference is a server-side, data-only component within Hytale's Entity Component System (ECS). Its sole purpose is to create a persistent link between an entity (typically an NPC) and a specific *Spawn Beacon* world object. This class acts as a data-bearing tag, holding a reference to the beacon that governs the entity's respawn behavior.

By attaching this component to an entity, the core spawning logic is decoupled from the entity itself. A dedicated manager, the SpawningSystem, can query for all entities possessing a SpawnBeaconReference and manage their lifecycle without needing to know the entity's specific type.

The presence of a static CODEC field signifies that this component is designed for serialization. It is persisted as part of the entity's data within the EntityStore, ensuring that the link to a spawn beacon survives server restarts and chunk unloading. This is a critical component for maintaining world state and NPC behavior over long periods.

## Lifecycle & Ownership
- **Creation:** A SpawnBeaconReference instance is created and attached to an entity when that entity's spawn behavior is bound to a specific beacon. This typically occurs under two scenarios:
    1.  **World Generation:** Pre-placed NPCs defined in world data are loaded with this component already attached.
    2.  **Dynamic Spawning:** An entity is spawned by a system (e.g., a quest trigger or monster spawner) that is configured to use a specific spawn beacon. The system adds this component to the newly created entity.
- **Scope:** The component's lifetime is strictly tied to the entity it is attached to. It persists as long as the entity exists in the world's EntityStore.
- **Destruction:** The component is destroyed and garbage collected when its parent entity is permanently removed from the game world. It can also be removed programmatically by a game system if an entity's spawn behavior changes (e.g., it is no longer bound to a beacon).

## Internal State & Concurrency
- **State:** The component's state is mutable. It primarily consists of the inherited *reference* field from its parent class, SpawnReference. This field holds the identifier or direct link to the associated spawn beacon.
- **Thread Safety:** This component is **not thread-safe**. Like most ECS components in the Hytale engine, it is designed to be accessed and mutated exclusively by systems running on the main server game loop thread. Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. Any cross-thread interaction must be managed via thread-safe message queues that are processed on the main thread.

## API Surface
The public contract is minimal, focusing on its integration with the ECS framework rather than direct manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the SpawningPlugin registry. Essential for ECS queries. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used by the engine for entity duplication or templating. |

## Integration Patterns

### Standard Usage
Developers should not interact with this component directly. Instead, server-side systems query for entities that possess this component to implement game logic.

```java
// A hypothetical system that checks the status of beacon-linked entities
ComponentType<EntityStore, SpawnBeaconReference> type = SpawnBeaconReference.getComponentType();
EntityStore world = server.getUniverse().getWorld();

// Find all entities managed by spawn beacons
for (Entity entity : world.getEntitiesWith(type)) {
    SpawnBeaconReference beaconRef = entity.get(type);
    // Logic to process the entity based on its linked beacon...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Queries:** Never use `new SpawnBeaconReference()` to check for component existence. Always use the entity's `has()` or `get()` methods with the type obtained from `getComponentType()`.
- **State Mutation Outside of Spawning Systems:** Modifying the internal beacon reference from a system unrelated to spawning (e.g., a combat or physics system) violates separation of concerns and can lead to unpredictable respawn behavior. All state changes should be routed through the authoritative SpawningSystem.
- **Client-Side Awareness:** This is a server-authoritative component. Client-side code should never attempt to access or reason about its existence. Doing so creates a brittle dependency on server implementation details.

## Data Pipeline
This component is primarily data-at-rest. It becomes active within a data flow when a specific game event, such as an entity's death, triggers the spawning logic.

> Flow:
> Entity Death Event -> SpawningSystem -> Query for **SpawnBeaconReference** on dead entity -> Read Beacon ID from component -> World lookup for Beacon's position -> Respawn Entity at new location


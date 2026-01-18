---
description: Architectural reference for SpawnReference
---

# SpawnReference

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Component (Abstract Data)

## Definition
```java
// Signature
public abstract class SpawnReference implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnReference is a server-side data component that establishes a persistent link between an entity (typically an NPC) and its designated spawn marker entity. It is a fundamental building block for systems that manage NPC population, such as spawners or patrol behaviors, ensuring that an NPC retains a connection to its origin point.

Architecturally, this component does not contain active logic. Instead, it serves as a state container that is read and manipulated by higher-level entity systems. Its primary responsibility is to encapsulate an `InvalidatablePersistentRef`, a specialized reference designed to survive server restarts and chunk unloading/reloading cycles.

The core concept is **resilience**. If an NPC's spawn marker is temporarily unavailable (e.g., the chunk it resides in is unloaded), the `InvalidatablePersistentRef` becomes invalid. The SpawnReference component provides a timeout mechanism (`markerLostTimeoutCounter`) to allow systems to gracefully handle this temporary disconnection without immediately despawning the NPC. This prevents cascading entity loss during normal world streaming operations.

### Lifecycle & Ownership
- **Creation:** SpawnReference instances are not created directly, as the class is abstract. Concrete subclasses are instantiated and attached to an entity's component set, typically during the entity's initial spawning process by a spawner system. It is also created via deserialization from world storage, orchestrated by the `EntityStore` using the provided `BASE_CODEC`.
- **Scope:** The lifecycle of a SpawnReference is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the `EntityStore`. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** This component is highly mutable. Its primary state consists of the `reference` to the spawn marker and the `markerLostTimeoutCounter`. Both are expected to change during runtime as systems interact with the component and the world state evolves.
- **Thread Safety:** This component is **not thread-safe** and must not be considered as such. All interactions, including state reads and modifications, must be performed exclusively on the main server thread responsible for ticking the parent entity. Unsynchronized access from other threads will lead to race conditions and world corruption.

## API Surface
The public API is minimal, designed for state inspection and manipulation by trusted entity systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReference() | InvalidatablePersistentRef | O(1) | Returns the persistent reference to the spawn marker entity. |
| tickMarkerLostTimeoutCounter(float dt) | boolean | O(1) | Decrements the timeout counter. Returns true if the timeout has expired. |
| refreshTimeoutCounter() | void | O(1) | Resets the timeout counter to its maximum value, typically used when the spawn marker is found. |
| clone() | Component | O(N) | Abstract. Subclasses must implement this to support entity cloning operations. |

## Integration Patterns

### Standard Usage
The intended pattern involves an entity system querying an entity for this component, checking the validity of the reference, and then updating the timeout state accordingly.

```java
// Example within an NPC AI or management system
void processNpc(Entity npc, float deltaTime) {
    SpawnReference spawnRef = npc.getComponent(SpawnReference.class);
    if (spawnRef == null) {
        // This NPC is not managed by a spawner
        return;
    }

    if (spawnRef.getReference().isValid()) {
        // Marker is loaded and valid, keep the NPC alive
        spawnRef.refreshTimeoutCounter();
    } else {
        // Marker is lost (e.g., chunk unloaded).
        // If the timeout expires, the NPC should be despawned.
        if (spawnRef.tickMarkerLostTimeoutCounter(deltaTime)) {
            world.removeEntity(npc);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Modification:** Do not directly access or modify the internal `reference` or `markerLostTimeoutCounter` fields. Use the provided public methods to ensure state transitions are handled correctly.
- **Ignoring Timeout:** Failing to call `tickMarkerLostTimeoutCounter` for NPCs with an invalid reference will result in orphaned entities that never despawn, leading to memory leaks and overpopulation.
- **Asynchronous Access:** Reading the reference validity or ticking the counter from a separate thread is strictly forbidden and will break server state.

## Data Pipeline
SpawnReference primarily participates in the entity serialization and runtime logic pipelines.

**Serialization Flow (Saving):**
> Entity is saved -> `EntityStore` serializes components -> `BASE_CODEC` is used for **SpawnReference** -> `InvalidatablePersistentRef` is written to disk storage.

**Runtime Logic Flow (Entity Update):**
> Flow:
> Server Tick -> Entity Update Loop -> NPC Management System -> Gets **SpawnReference** from Entity -> Checks `reference.isValid()` -> If invalid, calls `tickMarkerLostTimeoutCounter()` -> If timeout expires, triggers Despawn Action.


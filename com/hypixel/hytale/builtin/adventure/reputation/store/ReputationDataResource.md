---
description: Architectural reference for ReputationDataResource
---

# ReputationDataResource

**Package:** com.hypixel.hytale.builtin.adventure.reputation.store
**Type:** Data Component

## Definition
```java
// Signature
public class ReputationDataResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The ReputationDataResource is a data-holding component that encapsulates an entity's reputation with various factions. In Hytale's architecture, a Resource is a serializable data structure that can be attached to an EntityStore, which represents the persistent state of an in-game entity.

This class serves as a simple data container. Its primary responsibility is to hold a map of faction identifiers (String) to reputation values (integer). It does not contain any game logic itself. Instead, it is created, read, and modified by higher-level game systems, such as a FactionSystem or QuestSystem.

A critical feature is the static CODEC field. This is a declarative definition of how to serialize and deserialize an instance of ReputationDataResource. The engine's persistence layer uses this CODEC to save the component to disk when an entity is unloaded and to reconstruct it when the entity is loaded back into the world. This pattern is fundamental to Hytale's data-oriented component design.

## Lifecycle & Ownership
- **Creation:** An instance of ReputationDataResource is created under two primary conditions:
    1.  **Deserialization:** The most common path. When an entity is loaded from storage, the persistence engine uses the public CODEC to instantiate and populate the resource from saved data.
    2.  **Programmatic Instantiation:** A game system may create a new ReputationDataResource and attach it to an entity that does not yet have one, effectively enabling reputation tracking for that entity.
- **Scope:** The lifecycle of a ReputationDataResource is strictly tied to the EntityStore it is attached to. It exists in memory only as long as the parent entity is loaded in the world.
- **Destruction:** The resource is marked for garbage collection when its parent EntityStore is unloaded or destroyed, or if a system explicitly removes the component from the entity.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The core of the component is the reputationStats field, an Object2IntOpenHashMap that is expected to be modified during gameplay. The clone method performs a deep copy of this map, ensuring that cloned instances have independent state.
- **Thread Safety:** This class is **not thread-safe**. The underlying map, Object2IntOpenHashMap, is not synchronized. All reads and writes to this resource must be performed on the thread that owns the parent entity, which is typically the main server or world thread.

    **Warning:** Accessing or modifying a ReputationDataResource from multiple threads without explicit, external locking will result in race conditions, data corruption, and ConcurrentModificationExceptions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReputationStats() | Object2IntMap<String> | O(1) | Returns a direct reference to the internal reputation map. |
| clone() | Resource<EntityStore> | O(N) | Creates a new instance with a deep copy of the reputation map. N is the number of factions. |

## Integration Patterns

### Standard Usage
The intended use is for a game system to retrieve this component from an entity, modify the reputation map directly, and allow the engine to handle persistence.

```java
// Example: A system for awarding reputation
EntityStore playerEntity = ...; // Obtain the target entity
ReputationDataResource reputation = playerEntity.getResource(ReputationDataResource.class);

if (reputation != null) {
    Object2IntMap<String> stats = reputation.getReputationStats();
    stats.addTo("Villagers", 10); // Grant 10 reputation with the "Villagers" faction
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived References:** Do not retrieve the reputation map via getReputationStats and hold a reference to it for an extended duration. The parent entity could be unloaded, or the component could be removed, leading to stale data or unexpected errors. Always re-fetch the component from the entity for each logical operation.
- **External Serialization:** Do not attempt to serialize this object using mechanisms other than its built-in CODEC, such as standard Java serialization. Doing so will bypass the engine's data format and versioning, likely leading to data loss or world corruption.

## Data Pipeline
This component is a passive participant in the engine's entity persistence pipeline. It defines the "what" and "how" of its data, but the engine drives the process.

> **Serialization Flow (Saving):**
> Game System modifies reputation -> World Save Event -> EntityStore Persistence -> **ReputationDataResource.CODEC** serializes map -> Binary Data -> Disk
>
> **Deserialization Flow (Loading):**
> Disk -> Binary Data -> EntityStore Hydration -> **ReputationDataResource.CODEC** deserializes map -> New ReputationDataResource instance -> Attached to EntityStore


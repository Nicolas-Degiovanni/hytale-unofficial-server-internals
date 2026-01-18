---
description: Architectural reference for StoredFlock
---

# StoredFlock

**Package:** com.hypixel.hytale.server.flock
**Type:** Transient Data Model

## Definition
```java
// Signature
public class StoredFlock {
```

## Architecture & Concepts

The StoredFlock class is a specialized data container designed for the serialization and persistence of a group of related entities, referred to as a "flock". It serves as a critical bridge between the live, in-memory state of entities within the game world and their serialized, on-disk representation.

Its primary function is to manage the lifecycle of entity groups during world chunk unloading and loading. When a chunk containing a managed group of NPCs (for example, those associated with a spawn marker) is unloaded, the StoredFlock captures the state of each entity, removes them from the active world, and holds their data in a serializable format. When the chunk is reloaded, the StoredFlock is responsible for restoring these entities back into the world, preserving their components and data.

This mechanism is essential for maintaining the integrity of pre-defined or dynamically created entity groups across server sessions and world save/load cycles. It directly integrates with the server's Entity-Component-System (ECS) via the `Store`, `Holder`, and `Ref` abstractions.

### Lifecycle & Ownership
-   **Creation:** A StoredFlock instance is created under two conditions:
    1.  **Programmatically:** By a higher-level system, such as a spawner component, that needs to prepare a group of live entities for unloading.
    2.  **Deserialization:** By the Hytale `CODEC` system when loading world data from persistent storage. The static `CODEC` field defines how to reconstruct the object from its saved state.
-   **Scope:** The lifetime of a StoredFlock is bound to its owner. It is typically a field within another component (e.g., a `SpawnMarkerComponent`). It persists as long as its owning component exists, whether in memory or in a serialized save file. It is not a global or session-scoped object.
-   **Destruction:** The object is eligible for garbage collection when its owning component is destroyed. The `clear` method nullifies its internal data, and the `restoreNPCs` method implicitly clears the data after execution, making it a single-use object for restoration.

## Internal State & Concurrency
-   **State:** StoredFlock is a mutable, stateful object. Its core state is the `members` array, which holds the serialized `Holder<EntityStore>` representations of the flock's entities. This array is directly manipulated by the `storeNPCs`, `restoreNPCs`, and `clear` methods.
-   **Thread Safety:** This class is **not thread-safe**. All methods perform unsynchronized, direct modifications to the internal `members` array. All interactions with a StoredFlock instance must be performed on the main server thread to prevent data corruption and race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| storeNPCs(refs, store) | void | O(N) | Serializes a list of live entity references (`Ref`) into `Holder` objects and stores them internally. Removes the entities from the active world `Store`. |
| hasStoredNPCs() | boolean | O(1) | Returns true if the internal `members` array is not null, indicating it holds entity data. |
| restoreNPCs(output, store) | void | O(N) | Deserializes the internal `Holder` objects back into live entities, adding them to the world `Store`. Populates the output list with new `Ref`s. Clears internal state upon completion. |
| clear() | void | O(1) | Nullifies the internal `members` array, releasing all references to stored entity data. |
| clone() | StoredFlock | O(N) | Performs a deep copy of the StoredFlock, including a deep copy of the internal `Holder` array. |

## Integration Patterns

### Standard Usage

The typical pattern involves an owning system that manages a group of entities. On a world unload event, the owner creates a StoredFlock and populates it. On a world load event, it uses the StoredFlock to restore the entities.

```java
// Conceptual example during an unload event
List<Ref<EntityStore>> liveNpcs = ... // Get references to live NPCs
StoredFlock flockState = new StoredFlock();
flockState.storeNPCs(liveNpcs, world.getStore(EntityStore.class));
// The 'flockState' object is now ready to be serialized with its owning component.

// Conceptual example during a load event
StoredFlock loadedFlockState = ... // Deserialized from storage
List<Ref<EntityStore>> restoredNpcs = new ObjectArrayList<>();
loadedFlockState.restoreNPCs(restoredNpcs, world.getStore(EntityStore.class));
// The 'restoredNpcs' list now contains references to the newly created entities.
// The 'loadedFlockState' is now empty.
```

### Anti-Patterns (Do NOT do this)
-   **Reusing after Restore:** Do not attempt to call `restoreNPCs` multiple times on the same instance. The method clears its internal state after the first successful execution. Subsequent calls will have no effect.
-   **Concurrent Modification:** Do not access a StoredFlock instance from multiple threads. All operations must be synchronized externally, typically by ensuring they only run on the main server thread.
-   **State Assumption after Store:** After calling `storeNPCs`, the original `Ref` objects passed into the method will become invalid, as the underlying entities have been removed from the world. Do not attempt to use them further.

## Data Pipeline

The StoredFlock acts as a state transition manager in the entity persistence pipeline.

> **Serialization Flow (Unloading):**
> `List<Ref<EntityStore>>` (Live Entities) -> `storeNPCs()` -> **StoredFlock** (Contains `Holder<EntityStore>[]`) -> `CODEC` -> Persistent Storage

> **Deserialization Flow (Loading):**
> Persistent Storage -> `CODEC` -> **StoredFlock** (Contains `Holder<EntityStore>[]`) -> `restoreNPCs()` -> `List<Ref<EntityStore>>` (Live Entities)


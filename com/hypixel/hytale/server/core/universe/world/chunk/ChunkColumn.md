---
description: Architectural reference for ChunkColumn
---

# ChunkColumn

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Data Component

## Definition

```java
// Signature
@Deprecated
public class ChunkColumn implements Component<ChunkStore> {
```

## Architecture & Concepts

The ChunkColumn is a server-side data component that represents a complete vertical slice of the game world, from bedrock to the sky limit. It acts as a container for individual `Chunk` sections, which are 16x16x16 block volumes. This class is a fundamental unit of world data management, but it is critical to note its **@Deprecated** status, indicating it is part of a legacy system and should not be used for new feature development.

Its core architectural pattern is a **dual-state representation** of world data:

1.  **Dehydrated State (Holders):** The `sectionHolders` field stores an array of `Holder` objects. A `Holder` is a lightweight, serializable representation of a component. This state is used when the `ChunkColumn` is first loaded from disk or received over the network. It contains all necessary data but is not yet "live" in the game simulation.

2.  **Hydrated State (Refs):** The `sections` field stores an array of `Ref` objects. A `Ref` is a direct, in-memory reference to a live `ChunkStore` component. This state is used for active gameplay, allowing for high-performance block reads and writes by the game simulation thread.

The static `CODEC` field is the engine that facilitates the transition between these two states. It defines the logic for serializing live `Ref` data into storable `Holder` data and deserializing `Holder` data back into a `ChunkColumn` instance.

## Lifecycle & Ownership

-   **Creation:** A `ChunkColumn` is never instantiated directly by game logic. It is created by the world persistence layer during chunk loading. The `ChunkColumn.CODEC` is invoked to deserialize raw data from a `ChunkStore` into a new `ChunkColumn` object, populating its `sectionHolders` field.

-   **Scope:** An instance of `ChunkColumn` exists for as long as its corresponding world column is loaded in server memory. Its lifecycle is strictly managed by the server's `WorldManager` or an equivalent chunk-caching system.

-   **Destruction:** The object is eligible for garbage collection when the server unloads the chunk column, typically because no players are within its vicinity. The `WorldManager` is responsible for dropping all references to the object.

## Internal State & Concurrency

-   **State:** The `ChunkColumn` is a highly mutable state container. It functions as a cache for both serialized and live chunk data. Its internal state transitions from dehydrated (`sectionHolders` is populated) to hydrated (`sections` is populated) as the world system prepares it for active simulation.

-   **Thread Safety:** **This class is not thread-safe.** All interactions with a `ChunkColumn` instance, especially modification of its sections, must be performed on the main server thread. The internal arrays are exposed via getters, and unsynchronized access from multiple threads will lead to world corruption, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSection(int section) | Ref<ChunkStore> | O(1) | Retrieves a single, live chunk section. Returns null if out of bounds. |
| getSections() | Ref<ChunkStore>[] | O(1) | Retrieves the entire array of live chunk sections. |
| getSectionHolders() | Holder<ChunkStore>[] | O(1) | Retrieves the array of dehydrated, serializable section holders. |
| takeSectionHolders() | Holder<ChunkStore>[] | O(1) | **State Transition.** Moves ownership of the section holders out of this object, returning them and nullifying the internal field. This is a destructive read. |
| putSectionHolders(holders) | void | O(1) | Sets the internal array of dehydrated section holders. |
| clone() | Component<ChunkStore> | O(N) | Performs a deep clone of the component, merging live and dehydrated data into a new set of holders. |

## Integration Patterns

### Standard Usage

The `ChunkColumn` is exclusively managed by the server's world-loading systems. Game logic should not interact with its lifecycle. The typical internal flow involves deserialization, hydration, and eventual serialization.

```java
// 1. Deserialization (Handled by ChunkStore)
// ChunkColumn column = CODEC.decode(data);

// 2. Hydration (Handled by WorldManager)
// The world system takes the dehydrated holders to create live sections.
Holder<ChunkStore>[] holders = column.takeSectionHolders();
if (holders != null) {
    for (int i = 0; i < holders.length; i++) {
        // Logic to convert Holder into a live Ref and place it in column.sections
    }
}

// 3. Gameplay Access (Example system)
Ref<ChunkStore> section = column.getSection(5);
// ... modify blocks in the live section
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new ChunkColumn()`. The object's state is meaningless without being populated by the `CODEC` from a data source.

-   **State Confusion:** Do not attempt to read from `getSection()` before the world system has completed the hydration process. The `sections` array will be empty or incomplete.

-   **Misusing takeSectionHolders:** Calling `takeSectionHolders` more than once or outside of the chunk hydration pipeline will result in lost data, as it nullifies the internal field.

-   **External Modification:** Do not modify the arrays returned by `getSections` or `getSectionHolders`. This breaks encapsulation and can lead to an inconsistent state between the live and serialized representations.

-   **Usage in New Code:** The `@Deprecated` annotation is a strict warning. This class should not be referenced in any new systems.

## Data Pipeline

The `ChunkColumn` serves as a critical transformation point between persistent storage and the live game world.

> **Loading Flow:**
> Disk Storage (`ChunkStore`) -> Deserialization via `CODEC` -> **ChunkColumn** (with `sectionHolders` populated) -> World System Hydration -> **ChunkColumn** (with `sections` populated) -> Live Game Simulation

> **Saving Flow:**
> Live Game Simulation -> **ChunkColumn** (with `sections` populated) -> Serialization via `CODEC` -> Disk Storage (`ChunkStore`)


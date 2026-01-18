---
description: Architectural reference for GeneratedEntityChunk
---

# GeneratedEntityChunk

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient Data Container

## Definition
```java
// Signature
public class GeneratedEntityChunk {
```

## Architecture & Concepts
The GeneratedEntityChunk serves as a high-level, intermediate representation for entities created during the world generation process. It functions as a **staging area** or **builder** that decouples the abstract logic of world generation (e.g., "place a bandit camp prefab at this offset") from the low-level, concrete data structure of an EntityChunk.

World generation systems operate on groups of entities, often defined in prefabs. These groups have a collective position, rotation, and a common origin identifier. Instead of requiring the generator to calculate the final world-space transform for every single entity within the prefab, the generator can simply add the entire group to a GeneratedEntityChunk.

The primary responsibility of this class is to perform the final transformation pass. When the toEntityChunk method is invoked, it iterates through all staged entity groups, applies the specified group-level offsets and rotations to each individual entity's TransformComponent, and injects a FromWorldGen component for tracking provenance. The result is a fully-realized EntityChunk ready for integration into the world's entity storage system.

## Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by world generation algorithms during the processing of a single world chunk. Each generation task that produces entities will create a new, clean instance.
-   **Scope:** The object's lifetime is extremely short and is strictly confined to the scope of a single chunk generation operation. It accumulates entity data as the generator runs.
-   **Destruction:** Once the toEntityChunk method is called and the resulting EntityChunk is consumed by the world system, the GeneratedEntityChunk instance holds no further value. It is immediately eligible for garbage collection and should not be held or reused.

## Internal State & Concurrency
-   **State:** The internal state consists of a mutable list of EntityWrapperEntry records. The class is stateful and designed to be built up via successive calls to the addEntities method. The records themselves are immutable.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within a dedicated world generation worker. Any concurrent access, especially simultaneous calls to addEntities and toEntityChunk, will result in undefined behavior and data corruption. External synchronization is required if multi-threaded access is unavoidable, though this would violate the intended design.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntities(offset, rotation, holders, id) | void | O(1) | Stages a group of entities with a shared transform and origin ID. This is the primary method for populating the container. |
| toEntityChunk() | EntityChunk | O(N\*M) | **CRITICAL:** Transforms the staged entity data into a concrete EntityChunk. This is a computationally intensive, one-way operation. N is the number of staged groups; M is the number of entities per group. |
| getEntities() | List | O(1) | Returns a direct reference to the internal list of staged entity groups. Modifying this list externally is strongly discouraged. |
| forEachEntity(consumer) | void | O(N) | Provides a safe way to iterate over the staged entity groups without exposing the internal list. |

## Integration Patterns

### Standard Usage
The class is used as a temporary builder during world generation. The generator creates an instance, populates it with entity groups from prefabs or other sources, and then finalizes it into an EntityChunk.

```java
// A world generator creates a new instance for its task
GeneratedEntityChunk generatedEntities = new GeneratedEntityChunk();

// As the generator places features, it adds entities
// Example: Placing a "ruins" prefab
generatedEntities.addEntities(ruinsOffset, ruinsRotation, ruinsEntityHolders, RUINS_ID);

// Example: Placing a "monster spawner" feature
generatedEntities.addEntities(spawnerOffset, spawnerRotation, spawnerEntityHolders, SPAWNER_ID);

// At the end of the process, transform it into a concrete chunk
EntityChunk finalEntityChunk = generatedEntities.toEntityChunk();

// The final chunk is now ready to be merged into the world
world.mergeEntityChunk(chunkPosition, finalEntityChunk);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not reuse a GeneratedEntityChunk instance across the generation of multiple world chunks. This will cause entities from one chunk to bleed into another. Always create a new instance for each distinct generation task.
-   **Holding References:** Do not maintain a reference to a GeneratedEntityChunk after calling toEntityChunk. Its purpose is fulfilled, and holding the reference prevents it from being garbage collected.
-   **External Modification:** Do not retrieve the internal list via getEntities and modify it directly. This bypasses the intended data structure and can lead to an inconsistent state.

## Data Pipeline
This class is a key step in the entity world generation pipeline, acting as the bridge between abstract placement and concrete instantiation.

> Flow:
> World Generator Logic -> **GeneratedEntityChunk.addEntities()** -> Internal List of EntityWrapperEntry -> **GeneratedEntityChunk.toEntityChunk()** -> EntityChunk -> World State Integration


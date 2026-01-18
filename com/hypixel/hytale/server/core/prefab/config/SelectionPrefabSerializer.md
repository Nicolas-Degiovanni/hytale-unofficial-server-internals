---
description: Architectural reference for SelectionPrefabSerializer
---

# SelectionPrefabSerializer

**Package:** com.hypixel.hytale.server.core.prefab.config
**Type:** Utility

## Definition
```java
// Signature
public class SelectionPrefabSerializer {
```

## Architecture & Concepts
The SelectionPrefabSerializer is a stateless, static utility class that serves as the primary translation layer between the in-memory representation of a game structure, a BlockSelection, and its persistent BSON format. This class is a critical component of the engine's data interchange and content creation pipeline, responsible for saving and loading player-created structures, world editor prefabs, and procedurally generated points of interest.

Its core architectural responsibility is to ensure data compatibility across different versions of the game. The serializer contains significant logic for handling multiple legacy prefab versions, performing on-the-fly data migration for block names, and correctly interpreting deprecated data structures. This makes it a cornerstone of the engine's forward and backward compatibility strategy for user-generated content.

The serializer acts as a boundary object, mediating between the live game world (represented by BlockSelection, Entities, and Components) and the serialized world state (BSON). It directly interfaces with core asset systems like BlockTypeAssetMap and Fluid asset maps to resolve string identifiers into runtime numeric IDs.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its constructor is private, enforcing a purely static usage pattern.
- **Scope:** As a static utility, its methods are available for the entire lifetime of the server application. It holds no state and therefore has no individual lifecycle.
- **Destruction:** Not applicable. The class is unloaded by the JVM upon application shutdown.

## Internal State & Concurrency
- **State:** The SelectionPrefabSerializer is entirely stateless. All fields are static final constants, making them immutable. All operations are performed on method-local variables or the provided arguments.

- **Thread Safety:** This class is designed to be thread-safe.
    - The `serialize` and `deserialize` methods are re-entrant and do not modify any shared mutable state.
    - They perform read-only operations against globally accessible, thread-safe asset maps.
    - The use of `ExtraInfo.THREAD_LOCAL` in legacy decoding paths is a clear architectural choice to manage per-thread context without locks, ensuring high performance in concurrent environments such as a multi-threaded world generation pipeline.

    **WARNING:** While the methods themselves are thread-safe, the caller must ensure that the `BlockSelection` object passed to `serialize` is not mutated by another thread during the serialization process.

## API Surface
The public contract consists of two primary static methods for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(doc) | BlockSelection | O(N) | Reconstructs a BlockSelection from a BSON document. N is the total number of blocks, fluids, and entities. Throws IllegalArgumentException for unsupported or invalid prefab versions. |
| serialize(prefab) | BsonDocument | O(N log N) | Converts a BlockSelection into a BSON document. N is the number of blocks or fluids, with the log factor from sorting. Always serializes to the latest format version (8). |

## Integration Patterns

### Standard Usage
This class should be used whenever a BlockSelection needs to be persisted to disk or transmitted over the network. The caller is responsible for handling the BSON data source and destination.

```java
// Deserialization from a BSON document
BsonDocument prefabData = loadPrefabFromFile("castle.bson");
BlockSelection castleSelection = SelectionPrefabSerializer.deserialize(prefabData);

// Serialization to a BSON document
BlockSelection structureToSave = worldEditor.getCurrentSelection();
BsonDocument outputBson = SelectionPrefabSerializer.serialize(structureToSave);
savePrefabToFile("castle.bson", outputBson);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance of SelectionPrefabSerializer via reflection or other means.
- **Ignoring Version Exceptions:** The `deserialize` method will throw an exception if it encounters a prefab version it cannot handle. This is a critical error that must be caught and managed gracefully, typically by informing the user that the content is incompatible.
- **Assuming Format Stability:** The internal BSON structure is not a public contract. Do not manually construct or parse the BSON document; always use the `serialize` and `deserialize` methods to ensure compatibility with future game updates.

## Data Pipeline
The SelectionPrefabSerializer is a key transformation step in the data pipeline for game structures.

**Deserialization (Loading a Prefab):**
> Flow:
> BSON Document (File System / Network) → **SelectionPrefabSerializer.deserialize** → In-Memory BlockSelection → World Placement Engine → Live Blocks & Entities in World

**Serialization (Saving a Prefab):**
> Flow:
> Live Blocks & Entities in World → World Editor Copy Operation → In-Memory BlockSelection → **SelectionPrefabSerializer.serialize** → BSON Document → File System / Network


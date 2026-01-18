---
description: Architectural reference for CaveNodeChildEntryJsonLoader
---

# CaveNodeChildEntryJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave
**Type:** Transient

## Definition
```java
// Signature
public class CaveNodeChildEntryJsonLoader extends JsonLoader<SeedStringResource, CaveNodeType.CaveNodeChildEntry> {
```

## Architecture & Concepts
The CaveNodeChildEntryJsonLoader is a specialized deserializer within the server's procedural world generation framework. Its sole responsibility is to translate a single JSON object, representing a potential child connection in a cave graph, into a fully-realized CaveNodeType.CaveNodeChildEntry runtime object.

This class is a cornerstone of the data-driven design for cave generation. It allows world designers to define complex procedural rules—such as which cave segments can connect, their relative orientation, and spawn probabilities—in static JSON assets. This loader interprets those assets and prepares them for the generation algorithm.

Architecturally, it functions as a transient, single-purpose factory. It is provided with a JSON snippet and a dependency on the CaveNodeTypeStorage, which acts as a registry for all loaded cave segment definitions. The loader uses this storage to resolve string identifiers (e.g., "large_cavern") into concrete CaveNodeType objects, effectively linking the cave graph together during the loading phase.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level loader, typically CaveNodeTypeJsonLoader, whenever it parses the *children* array within a cave node's JSON definition. A new CaveNodeChildEntryJsonLoader is instantiated for each element in that array.

- **Scope:** The object's lifetime is exceptionally brief. It exists only for the duration of a single call to its `load` method. Once the resulting CaveNodeChildEntry is returned, the loader instance has served its purpose and is immediately eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup requirements.

## Internal State & Concurrency
- **State:** This class is stateful, holding immutable references to the input JsonElement, a procedural SeedString, the data folder Path, and the CaveNodeTypeStorage registry. This state is provided at construction and is not modified during the object's lifecycle. The object does not perform any internal caching.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to operate within a single-threaded asset loading pipeline. All world generation asset parsing should be performed synchronously or be externally synchronized to prevent race conditions, particularly with the shared CaveNodeTypeStorage dependency.

## API Surface
The public contract is minimal, centered on the `load` method. The constructor is public for composition by other loaders but is not intended for direct use by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CaveNodeType.CaveNodeChildEntry | O(N) | Deserializes the configured JSON into a CaveNodeChildEntry object. N is the number of nodes in the "Node" array. Throws IllegalArgumentException on malformed or invalid data. |

## Integration Patterns

### Standard Usage
This class is an internal component of the world generation system and is not intended for direct invocation. The following conceptual example demonstrates how a parent loader would use it.

```java
// Conceptual usage within a parent CaveNodeTypeJsonLoader
CaveNodeTypeStorage storage = getCaveNodeTypeStorage();
JsonArray childrenJson = parentJson.get("children").getAsJsonArray();

for (JsonElement childEntryJson : childrenJson) {
    // A new, short-lived loader is created for each child definition
    CaveNodeChildEntryJsonLoader loader = new CaveNodeChildEntryJsonLoader(seed, dataFolder, childEntryJson, storage);
    CaveNodeType.CaveNodeChildEntry entry = loader.load();
    
    // The resulting entry is added to the parent node's definition
    parentNode.addChildEntry(entry);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class directly in game logic. It is an implementation detail of the asset loading pipeline. Interacting with the world generation system should be done through higher-level APIs.

- **Instance Re-use:** Do not attempt to re-use a loader instance. Each instance is configured for a specific JSON element and is not designed to be repurposed. Create a new loader for each distinct JSON object to be parsed.

- **Concurrent Loading:** Do not call the `load` method from multiple threads on the same instance or on different instances that share a non-thread-safe CaveNodeTypeStorage.

## Data Pipeline
This loader acts as a specific transformation step within the broader cave asset loading pipeline. It consumes a raw JSON structure and produces a structured, in-memory object ready for use by the procedural generation engine.

> Flow:
> Cave Definition JSON File -> `CaveNodeTypeJsonLoader` -> **CaveNodeChildEntryJsonLoader** -> `CaveNodeType.CaveNodeChildEntry` (In-Memory Object) -> Cave Generation Algorithm


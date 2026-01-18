---
description: Architectural reference for EmptyLineCaveNodeShapeGeneratorJsonLoader
---

# EmptyLineCaveNodeShapeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class EmptyLineCaveNodeShapeGeneratorJsonLoader extends CaveNodeShapeGeneratorJsonLoader {
```

## Architecture & Concepts
The EmptyLineCaveNodeShapeGeneratorJsonLoader is a specialized component within the server's procedural world generation framework. Its primary function is to act as a deserializer, translating a specific JSON configuration structure into a live, executable Java object: the EmptyLineCaveNodeShapeGenerator.

This class embodies the Strategy pattern for data loading. A higher-level orchestrator, responsible for parsing world generation files, identifies a JSON object defining an "EmptyLine" cave shape and delegates the parsing task to this specific loader. This isolates the logic for handling one particular shape type, promoting modularity and extensibility within the world generation pipeline.

It further demonstrates a compositional design by delegating the parsing of a numeric range for the cave's length to a more generic utility, the DoubleRangeJsonLoader. This prevents code duplication and centralizes the logic for handling common data structures found in procedural content files.

## Lifecycle & Ownership
- **Creation:** Instantiated by a parent factory or registry within the world generation system. This occurs when the system encounters a JSON object that is designated to define an "EmptyLine" cave shape. The caller provides the specific JSON snippet, a procedural seed, and the data folder context required for the load operation.
- **Scope:** Extremely short-lived. The lifecycle of this object is confined to a single load operation. It is created, its load method is immediately invoked, and the loader object itself is then eligible for garbage collection. The *result* of the load operation is what persists, not the loader.
- **Destruction:** The object is garbage collected immediately after the load method returns its result to the caller. It holds no persistent state or external resources that require manual cleanup.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal fields for the seed, data folder, and JSON element are set during construction and are not modified thereafter. The class is stateless in the sense that it does not change its internal configuration across multiple method calls.
- **Thread Safety:** This class is not designed for concurrent access. It is inherently thread-safe for its intended use case because it is transient and operates on data passed in via its constructor. Multiple instances of this loader can be safely used on different threads to parse different JSON objects simultaneously, as they do not share any state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | EmptyLineCaveNodeShape.EmptyLineCaveNodeShapeGenerator | O(N) | Deserializes the configured JSON data into a generator instance. N is the size of the JSON fragment being parsed. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is an internal component of the world generation system, invoked by a higher-level content loader.

```java
// This class is typically invoked by a higher-level procedural content loader.
// Assume 'parentLoader' is responsible for routing JSON to the correct loader.

JsonElement shapeJson = ...; // A JSON object defining an EmptyLine shape
SeedString seed = ...;
Path dataFolder = ...;

// The parent loader instantiates this class for a specific deserialization task.
EmptyLineCaveNodeShapeGeneratorJsonLoader loader = new EmptyLineCaveNodeShapeGeneratorJsonLoader(seed, dataFolder, shapeJson);

// The loader is used immediately to produce the runtime object.
EmptyLineCaveNodeShape.EmptyLineCaveNodeShapeGenerator generator = loader.load();

// The 'loader' instance is now out of scope and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and re-use an instance of this loader. It is configured for a single, specific JsonElement. A new instance must be created for each distinct JSON object that needs to be parsed.
- **State Modification:** Do not attempt to modify the internal state of this loader after construction via reflection or other means. It is designed to be a single-use, immutable parser.

## Data Pipeline
This loader sits at a specific point in the data flow that transforms declarative data files into active components of the world generator.

> Flow:
> WorldGen JSON File -> Master Procedural Loader -> **EmptyLineCaveNodeShapeGeneratorJsonLoader** -> EmptyLineCaveNodeShape.EmptyLineCaveNodeShapeGenerator -> Cave Generation System


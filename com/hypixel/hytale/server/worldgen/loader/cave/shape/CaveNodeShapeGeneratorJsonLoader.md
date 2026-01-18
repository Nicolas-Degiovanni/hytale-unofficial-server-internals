---
description: Architectural reference for CaveNodeShapeGeneratorJsonLoader
---

# CaveNodeShapeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Utility / Abstract Base Class

## Definition
```java
// Signature
public abstract class CaveNodeShapeGeneratorJsonLoader extends JsonLoader<SeedStringResource, CaveNodeShapeEnum.CaveNodeShapeGenerator> {
```

## Architecture & Concepts
The CaveNodeShapeGeneratorJsonLoader is a foundational component within the server-side procedural world generation pipeline. It serves as an abstract factory, establishing a standardized contract for translating declarative JSON configuration files into executable Java objects.

Its primary role is to act as a type-safe bridge between on-disk world generation assets and the in-memory representation of cave generation logic. By extending the generic JsonLoader, it inherits a robust framework for handling file I/O and basic JSON parsing, while specializing the output to a CaveNodeShapeEnum.CaveNodeShapeGenerator.

This class is not intended for direct use. Instead, it defines the template that concrete loader implementations must follow. This architectural pattern ensures that all cave shape definitions, regardless of their specific JSON structure, are loaded and instantiated consistently throughout the world generation engine.

### Lifecycle & Ownership
- **Creation:** A concrete subclass of CaveNodeShapeGeneratorJsonLoader is instantiated by a higher-level factory or registry during the server's world generation initialization phase. It is created on-demand to process a specific JsonElement.
- **Scope:** Transient. The lifecycle of a loader instance is extremely short, confined to a single `load` operation. It is created, its primary method is invoked, and it is immediately eligible for garbage collection.
- **Destruction:** Handled by the Java Garbage Collector. As these are stateless, short-lived objects, there is no manual cleanup required.

## Internal State & Concurrency
- **State:** Effectively immutable after construction. The loader holds references to the world seed, data folder path, and the specific JsonElement it is tasked with parsing. This state is used as input for the loading process and is not modified.
- **Thread Safety:** **Not thread-safe.** Loader instances are designed for single-threaded execution within the world generation bootstrap sequence. Sharing a single instance across multiple threads will result in unpredictable behavior and potential resource contention. Each loading task must create its own dedicated loader instance.

## API Surface
The public API is inherited from the parent JsonLoader class. The constructor is the only symbol defined directly on this class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CaveNodeShapeGeneratorJsonLoader(seed, dataFolder, json) | constructor | O(1) | Constructs the loader with the necessary context for a parsing operation. |
| load() | CaveNodeShapeEnum.CaveNodeShapeGenerator | O(N) | *Inherited.* The primary entry point. Parses the internal JsonElement and returns a fully configured generator object. N is the number of tokens in the JSON structure. |

## Integration Patterns

### Standard Usage
A concrete implementation of this class is typically invoked by a factory that maps a JSON `type` field to the correct loader class. The caller provides the JSON data and receives a ready-to-use generator object.

```java
// Note: This example uses a hypothetical concrete implementation for clarity.
// The worldgen system would typically automate this selection.

JsonElement shapeDefinition = readCaveShapeConfig("my_cave_shape.json");
SeedString<SeedStringResource> worldSeed = ...;
Path dataPath = ...;

// The factory would select the correct loader based on the JSON content
CaveNodeShapeGeneratorJsonLoader loader = new MyConcreteShapeLoader(worldSeed, dataPath, shapeDefinition);

// The loaded object is now ready for use in the world generator
CaveNodeShapeEnum.CaveNodeShapeGenerator generator = loader.load();
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse loader instances. They are lightweight and tied to specific input JSON. Caching provides no performance benefit and risks state pollution if the design were to change.
- **Direct Instantiation:** Avoid `new MyConcreteShapeLoader()` in game logic. The world generation system uses a registry or factory to manage loaders. Bypassing this system can lead to improperly configured or uninitialized components.

## Data Pipeline
This class is a critical transformation step in the world generation asset pipeline. It converts structured text data into an executable object.

> Flow:
> JSON File on Disk -> Resource Manager -> JsonElement -> **CaveNodeShapeGeneratorJsonLoader** -> CaveNodeShapeGenerator Instance -> Cave Generation Algorithm


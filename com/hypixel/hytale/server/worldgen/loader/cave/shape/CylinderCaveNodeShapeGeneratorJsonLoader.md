---
description: Architectural reference for CylinderCaveNodeShapeGeneratorJsonLoader
---

# CylinderCaveNodeShapeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class CylinderCaveNodeShapeGeneratorJsonLoader extends CaveNodeShapeGeneratorJsonLoader {
```

## Architecture & Concepts
The CylinderCaveNodeShapeGeneratorJsonLoader is a specialized factory component within the server-side procedural world generation framework. Its sole responsibility is to deserialize a specific JSON configuration structure into a fully-realized CylinderCaveNodeShapeGenerator instance.

This class operates as a concrete implementation within a larger, polymorphic loading system. A higher-level orchestrator, responsible for parsing world generation data files, identifies a JSON object defining a "cylinder" cave shape and delegates the instantiation logic to this specific loader. This pattern decouples the core world generation engine from the specific data formats used to define its components, allowing for extensibility.

It follows a principle of composition by delegating the parsing of ranged numeric values (like radius and length) to the DoubleRangeJsonLoader utility. This isolates the logic for handling complex data types and promotes reuse across different loader implementations.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a parent factory or data registry during the world generation bootstrap phase. A new instance is created for each distinct cylinder cave shape definition encountered in the source JSON files. It is **not** a singleton.

- **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of the `load` method call. Once it has produced the CylinderCaveNodeShapeGenerator, its purpose is fulfilled, and it becomes eligible for garbage collection.

- **Destruction:** Managed automatically by the Java Garbage Collector. There are no native resources or persistent connections requiring manual cleanup.

## Internal State & Concurrency
- **State:** The internal state of this class consists of the `seed`, `dataFolder`, and `json` element provided at construction. This state is treated as immutable and is used exclusively as input for the `load` operation. The class performs no internal caching or state modification.

- **Thread Safety:** This class is **not thread-safe** and is intended for single-threaded access. It is designed to be used as a transient tool within the world generation pipeline, which is expected to manage its own threading model. Do not share instances of this loader across threads.

## API Surface
The public contract is minimal, focused entirely on the factory pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CylinderCaveNodeShapeGenerator | O(1) | Deserializes the internal JSON and constructs a new generator. This is the primary entry point. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay or system developers. It is invoked by the world generation data loading framework. The typical pattern involves a higher-level system reading a JSON file, identifying the shape type, and dispatching to the appropriate loader.

```java
// Pseudo-code for the calling framework
CaveNodeShapeGeneratorJsonLoader loader = factory.getLoaderFor(jsonDefinition); // Returns an instance of this class
CaveNodeShapeGenerator generator = loader.load();
caveSystem.registerGenerator(generator);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CylinderCaveNodeShapeGeneratorJsonLoader()`. The world generation framework is responsible for creating loader instances with the correct context, including the procedural seed and data paths.

- **Instance Caching:** Do not retain a reference to this loader after calling `load`. Each instance is tied to a specific JSON element and is not designed for reuse.

- **State Mutation:** Do not attempt to modify the `JsonElement` passed to the constructor while the loader is active. The input data is expected to be stable during the `load` operation.

## Data Pipeline
This component acts as a transformation step, converting declarative data from a file into an executable object in memory.

> Flow:
> World Gen Asset (JSON file) -> Engine File Parser -> **CylinderCaveNodeShapeGeneratorJsonLoader** -> CylinderCaveNodeShapeGenerator (In-memory object) -> Procedural Cave Generation System


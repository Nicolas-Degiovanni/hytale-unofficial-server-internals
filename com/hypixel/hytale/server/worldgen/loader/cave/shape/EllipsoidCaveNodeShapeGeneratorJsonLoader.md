---
description: Architectural reference for EllipsoidCaveNodeShapeGeneratorJsonLoader
---

# EllipsoidCaveNodeShapeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Transient Factory

## Definition
```java
// Signature
public class EllipsoidCaveNodeShapeGeneratorJsonLoader extends CaveNodeShapeGeneratorJsonLoader {
```

## Architecture & Concepts
The EllipsoidCaveNodeShapeGeneratorJsonLoader is a specialized factory component within the server-side procedural world generation pipeline. Its sole responsibility is to deserialize a specific JSON configuration object and construct an instance of EllipsoidCaveNodeShape.EllipsoidCaveNodeShapeGenerator.

This class operates as part of a larger, polymorphic loading framework. A master configuration loader determines the desired cave node shape from a JSON definition (e.g., by reading a "type" field) and delegates the construction of that shape generator to the appropriate loader subclass. This class handles the "ellipsoid" shape type.

It is deeply integrated with the deterministic procedural library, using a SeedString to ensure that cave generation is repeatable for a given world seed. It further composes other loaders, such as DoubleRangeJsonLoader, to parse complex data types from the configuration, promoting a modular and reusable loader architecture.

## Lifecycle & Ownership
-   **Creation:** This object is instantiated dynamically by a higher-level factory or registry during the world generation asset loading phase. It is never created directly by game logic. The parent loader reads a configuration file, identifies the need for an ellipsoid shape, and constructs this loader to process the relevant JSON section.

-   **Scope:** Extremely short-lived and transient. An instance of this class exists only for the duration of a single `load` operation. It is created, its `load` method is called once, and it is then immediately discarded.

-   **Destruction:** The object becomes eligible for garbage collection as soon as the `load` method returns its result. There are no persistent references to the loader instance itself.

## Internal State & Concurrency
-   **State:** The loader's state (seed, data folder, and JSON element) is provided via its constructor and is treated as immutable. The class does not contain any internal mutable state or caches. The `load` method is a pure function of its initial constructor arguments.

-   **Thread Safety:** This class is not designed for concurrent access and requires no internal synchronization. Its transient lifecycle inherently prevents race conditions. It is safe to use within a multi-threaded loading screen, provided that each thread processes different configuration files and creates its own loader instances.

## API Surface
The public contract is minimal, focused entirely on the factory pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | EllipsoidCaveNodeShape.EllipsoidCaveNodeShapeGenerator | O(1) | Parses the JSON data and returns a fully configured generator. Throws NullPointerException if required radius keys are missing. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is invoked by the world generation framework. The following example is conceptual, illustrating how a parent loader would use this component.

```java
// Conceptual example of a parent loader's logic
JsonElement shapeConfig = getShapeConfigFromJson(); // Assume this is the "ellipsoid" config
SeedString seed = world.getSeed().append(".caves");
Path dataFolder = world.getDataFolder();

// The framework instantiates and uses the loader in one go
CaveNodeShapeGenerator generator = new EllipsoidCaveNodeShapeGeneratorJsonLoader(seed, dataFolder, shapeConfig).load();

// The generator is now ready for use in the worldgen pipeline
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually construct this class with `new`. The world generation system is responsible for selecting and creating the correct loader based on configuration data.
-   **Instance Re-use:** Do not hold a reference to this loader and call `load` multiple times. It is a single-use object and should be discarded after the generator has been created.
-   **Configuration Fallback:** Do not rely on this loader to provide default values if the JSON is malformed. It enforces a strict contract and will fail fast by throwing an exception if required keys like `Radius` or `RadiusX/Y/Z` are not present.

## Data Pipeline
This loader acts as a transformation step in the broader world generation configuration pipeline. It converts structured text data into a functional in-memory object.

> Flow:
> World Generation JSON File -> Master Config Loader -> **EllipsoidCaveNodeShapeGeneratorJsonLoader** -> EllipsoidCaveNodeShapeGenerator Instance -> Cave Generation Algorithm


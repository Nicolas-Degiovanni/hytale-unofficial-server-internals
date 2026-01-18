---
description: Architectural reference for CoordinateRotatorJsonLoader
---

# CoordinateRotatorJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient / Factory

## Definition
```java
// Signature
public class CoordinateRotatorJsonLoader<K extends SeedResource> extends JsonLoader<K, CoordinateRotator> {
```

## Architecture & Concepts
The CoordinateRotatorJsonLoader is a specialized factory component within the procedural generation library's data loading system. Its sole responsibility is to deserialize a specific JSON object structure into a concrete `CoordinateRotator` or `CoordinateOriginRotator` instance.

Architecturally, this class embodies the Strategy pattern for data parsing. It is one of many `JsonLoader` implementations, each designed to handle a specific, well-defined data schema within a larger asset file. This decouples the high-level asset management system from the low-level details of parsing geometric transformation data. It acts as a translation layer between the declarative, on-disk JSON format and the engine's in-memory representation of a coordinate system rotation, which is a fundamental operation in procedural world generation.

## Lifecycle & Ownership
- **Creation:** An instance is created dynamically by a higher-level configuration or asset parser when it encounters a JSON object designated to represent a rotation (e.g., a block with the key "Rotate"). It is not a managed service and should never be injected.
- **Scope:** The object's lifetime is ephemeral. It is designed to be single-use and exists only for the duration of the parsing operation. Its scope is typically confined to the method that instantiates it.
- **Destruction:** The object holds no external resources and is eligible for garbage collection immediately after the `load` method returns its result. There is no explicit cleanup required.

## Internal State & Concurrency
- **State:** The internal state consists of immutable references to the source `JsonElement` and associated metadata (`seed`, `dataFolder`) provided at construction. The object does not modify this state or cache any results.
- **Thread Safety:** This class is **not thread-safe** and is not intended for concurrent use. It is designed to be instantiated, used, and discarded within a single-threaded asset loading pipeline. Sharing an instance across multiple threads will lead to unpredictable behavior. All operations should be confined to the thread that created the object.

## API Surface
The public contract is minimal, focused entirely on the single `load` operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CoordinateRotator | O(1) | Deserializes the source JSON into a rotation object. Returns a shared, static instance for no-op rotations as an optimization. |

## Integration Patterns

### Standard Usage
This class is an internal implementation detail of the asset pipeline and is not intended for direct use by gameplay systems. A parent loader is responsible for instantiating and invoking it.

```java
// Hypothetical usage within a parent asset parser
JsonElement rotationJson = parentJson.get("Rotate");
if (rotationJson != null) {
    // Delegate parsing to the specialized loader
    CoordinateRotatorJsonLoader<MySeed> loader = new CoordinateRotatorJsonLoader<>(seed, dataFolder, rotationJson);
    CoordinateRotator rotator = loader.load();
    
    // Use the resulting rotator object in the procedural system
    proceduralObject.setRotation(rotator);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of this loader for later use. It is stateless and designed to be created on-demand for a single parsing task.
- **Manual Construction:** Avoid manually constructing a `JsonElement` to pass to this loader. It is designed to operate on data fragments parsed from complete and valid asset files managed by the engine.

## Data Pipeline
This component is a single, critical step in the transformation of on-disk data into a usable in-memory object for the procedural generation engine.

> Flow:
> Asset File on Disk -> Engine JSON Parser -> `JsonElement` Fragment -> **CoordinateRotatorJsonLoader** -> `CoordinateRotator` Instance -> Procedural Placement System


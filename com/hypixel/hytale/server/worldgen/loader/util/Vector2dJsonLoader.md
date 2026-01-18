---
description: Architectural reference for Vector2dJsonLoader
---

# Vector2dJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient Utility

## Definition
```java
// Signature
public class Vector2dJsonLoader extends JsonLoader<SeedStringResource, Vector2d> {
```

## Architecture & Concepts
The Vector2dJsonLoader is a specialized component within the server's world generation framework. It serves a single, critical purpose: to translate a JSON data structure from a configuration file into a `Vector2d` game object. This class acts as a data-binding adapter, providing a robust and flexible bridge between human-readable configuration and the internal data types used by procedural generation algorithms.

It extends the generic JsonLoader, indicating its role as a concrete implementation of the **Strategy Pattern** for handling a specific data type. The loader is designed with flexibility in mind, capable of parsing multiple valid JSON representations for a 2D vector:
*   A JSON array with two numbers: `[10.5, -20.0]`
*   A JSON array with a single number, which is duplicated for both components: `[50]` becomes `(50, 50)`
*   A JSON object with explicit keys: `{ "X": 10.5, "Y": -20.0 }`
*   Null or empty values, which gracefully default to a zero vector `(0, 0)`.

This tolerance for multiple formats simplifies the authoring of world generation configuration files.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration parsing system whenever it encounters a field that requires a `Vector2d` value. It is instantiated with the specific `JsonElement` it is responsible for parsing.
- **Scope:** The object's lifetime is extremely short and ephemeral. It is designed to be single-use, existing only for the duration of the `load` method call.
- **Destruction:** It becomes eligible for garbage collection immediately after the `load` method returns its `Vector2d` result. It holds no persistent state or external references that would prolong its life.

## Internal State & Concurrency
- **State:** The internal state of the loader (the source `JsonElement`, `seed`, and `dataFolder`) is provided at construction and is treated as immutable. The `load` method does not modify this state; it is a pure transformation function.
- **Thread Safety:** This class is inherently thread-safe and re-entrant. Its immutable state and lack of side effects mean that it can be safely instantiated and used in multi-threaded world generation contexts without requiring any external locking or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | Vector2d | O(1) | Parses the configured JsonElement into a Vector2d instance. Returns a default zero vector if the source JSON is null, empty, or malformed. This method is non-blocking and computationally inexpensive. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is an internal utility invoked by the procedural generation library's core JSON parsing engine. A developer would typically define a `Vector2d` in a JSON file, and the framework would automatically use this loader behind the scenes.

A conceptual example of its internal invocation:
```java
// The framework has parsed a JsonElement from a config file
JsonElement vectorElement = someJsonObject.get("spawnOffset");

// It instantiates the loader for this specific element and immediately uses it
Vector2dJsonLoader loader = new Vector2dJsonLoader(seed, dataFolder, vectorElement);
Vector2d spawnOffset = loader.load();

// The loader instance is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse an instance of Vector2dJsonLoader. Each instance is tied to a specific `JsonElement` and is not designed for repeated use with different data.
- **Manual Instantiation:** Avoid calling `new Vector2dJsonLoader()` in game logic. The broader `procedurallib` framework is responsible for selecting the correct loader for a given data type. Rely on the framework's deserialization mechanisms.

## Data Pipeline
The class functions as a single, simple step in a larger data ingestion pipeline for world generation.

> Flow:
> Worldgen JSON File -> GSON Parser -> `JsonElement` -> **Vector2dJsonLoader** -> `Vector2d` Object -> Procedural Generation Algorithm


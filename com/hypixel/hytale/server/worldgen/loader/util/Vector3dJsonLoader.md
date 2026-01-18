---
description: Architectural reference for Vector3dJsonLoader
---

# Vector3dJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient

## Definition
```java
// Signature
public class Vector3dJsonLoader extends JsonLoader<SeedStringResource, Vector3d> {
```

## Architecture & Concepts
The Vector3dJsonLoader is a specialized, single-purpose deserializer within the server's world generation framework. Its primary function is to translate a generic JsonElement from a world generation asset file into a strongly-typed Vector3d object.

This class acts as a flexible data adapter, allowing world designers to specify 3D coordinates or vectors in multiple convenient JSON formats (object, array, or single value) without requiring changes to the core world generation code. It encapsulates the parsing logic, providing a clean separation between data definition and its programmatic use. It is a low-level utility, expected to be invoked by higher-level configuration loaders responsible for parsing entire asset files, such as biome or structure definitions.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a parent JsonLoader or configuration parser when it encounters a JSON field that must be resolved into a Vector3d. It is not managed by a dependency injection container or service registry.
- **Scope:** Extremely short-lived. The object's scope is strictly confined to the duration of a single deserialization task. It is created, its load method is invoked once, and it is then immediately discarded.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the load method returns its result. It holds no persistent state or external references, ensuring a clean and immediate cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the seed, data folder path, and the target JsonElement, is provided at construction and is treated as immutable for the object's lifetime. This class is stateless in the sense that it performs a pure transformation on its inputs without retaining any information post-execution.
- **Thread Safety:** This class is **not thread-safe** for shared instances. It is designed to be instantiated and used within a single thread of execution. While its immutable state makes it technically safe if a new instance is created per-thread, sharing a single instance across multiple threads is an unsupported and dangerous pattern. The world generation pipeline must ensure that each parsing task uses a new loader instance.

## API Surface
The public contract is minimal, focused exclusively on the deserialization task.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | Vector3d | O(1) | Deserializes the provided JsonElement into a Vector3d instance. Throws a fatal Error if the JSON structure is invalid or cannot be parsed. |

## Integration Patterns

### Standard Usage
This loader is intended to be used as a delegate by a more complex configuration parser. The parent parser identifies a JSON field and hands off the specific JsonElement to a new Vector3dJsonLoader for processing.

```java
// Hypothetical parent loader processing a config file
JsonObject config = ...; // Loaded from a file
JsonElement vectorData = config.get("spawnOffset");

// Delegate parsing to the specialized loader
Vector3dJsonLoader loader = new Vector3dJsonLoader(seed, dataFolder, vectorData);
Vector3d spawnOffset = loader.load();

// Use the resulting Vector3d in the world generation algorithm
world.setSpawnPoint(spawnOffset);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse instances of Vector3dJsonLoader. They are extremely lightweight and designed to be single-use. Reusing an instance with different data is not supported.
- **Defensive Error Handling:** Do not wrap the load call in a try-catch block for `Error`. A thrown `Error` indicates a critical, unrecoverable misconfiguration in the world generation assets. This is a fail-fast mechanism that must be addressed by fixing the source JSON file, not by handling it at runtime.

## Data Pipeline
The Vector3dJsonLoader is a single, focused step in the broader world generation asset loading pipeline.

> Flow:
> WorldGen JSON Asset -> GSON Parser -> JsonElement -> **Vector3dJsonLoader** -> Vector3d Instance -> World Generation Algorithm


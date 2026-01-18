---
description: Architectural reference for NoiseHeightThresholdInterpreterJsonLoader
---

# NoiseHeightThresholdInterpreterJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class NoiseHeightThresholdInterpreterJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseHeightThresholdInterpreter> {
```

## Architecture & Concepts
The NoiseHeightThresholdInterpreterJsonLoader is a specialized deserializer within the procedural generation framework. Its primary function is to translate a specific JSON configuration structure into a fully realized, in-memory NoiseHeightThresholdInterpreter object. This class acts as a critical bridge between the declarative, data-driven world generation definitions stored on disk and the imperative code that executes the generation logic.

This loader embodies the Composite and Factory patterns. It constructs a complex NoiseHeightThresholdInterpreter object by recursively invoking other, more granular loaders (NoisePropertyJsonLoader, HeightThresholdInterpreterJsonLoader) to build the constituent parts of the final object graph. The static method *shouldHandle* allows a parent factory or dispatcher to inspect a generic JsonObject and determine if this loader is the correct one to process it, enabling a flexible, data-driven loading system.

## Lifecycle & Ownership
-   **Creation:** This object is instantiated by a higher-level configuration loader or factory. The parent system identifies a relevant JSON object, typically by using the static *shouldHandle* method, and then creates a new instance of this loader, passing the specific JSON fragment and contextual data like the data folder path.
-   **Scope:** The lifecycle of a NoiseHeightThresholdInterpreterJsonLoader instance is extremely short and scoped to a single operation. It is created, its *load* method is called once, and it is then immediately discarded.
-   **Destruction:** The loader object becomes eligible for garbage collection as soon as the *load* method returns. The caller takes ownership of the returned NoiseHeightThresholdInterpreter, not the loader itself.

## Internal State & Concurrency
-   **State:** The loader is stateful, containing references to the input JSON, a seed, and a data folder path. This state is provided during construction and is treated as immutable for the object's lifetime. It performs no internal caching beyond the data it is initialized with.
-   **Thread Safety:** This class is **not thread-safe** and is intended for single-threaded use. It performs no internal locking or synchronization. It is expected that each deserialization task on each thread will use its own dedicated instance of the loader.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseHeightThresholdInterpreter | O(N) | The primary entry point. Parses the internal JSON and constructs the object graph. N is the number of thresholds. Throws IllegalStateException if required JSON keys (Noise, Thresholds, Keys) are missing. |
| shouldHandle(JsonObject) | static boolean | O(1) | A predicate used by factory systems to determine if this loader can process the given JsonObject. Checks for the existence of the "Thresholds" key. |

## Integration Patterns

### Standard Usage
This loader is not intended to be used directly by game logic developers. It is invoked by the core procedural asset loading system. A parent loader identifies the correct JSON snippet and dispatches it to this handler.

```java
// Conceptual usage by a parent factory
JsonObject configChunk = getSomeConfig();
NoiseHeightThresholdInterpreter interpreter = null;

if (NoiseHeightThresholdInterpreterJsonLoader.shouldHandle(configChunk)) {
    // The factory creates a new, short-lived loader instance for the operation
    interpreter = new NoiseHeightThresholdInterpreterJsonLoader<>(seed, dataFolder, configChunk, length).load();
}

// Use the resulting interpreter object in the world generator
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not retain an instance of this loader to call *load* multiple times. It is designed as a single-use, throwaway object.
-   **Manual Instantiation:** Avoid constructing this class manually with `new`. The procedural loading framework is responsible for its creation and providing the correct context (seed, dataFolder, json).
-   **Ignoring shouldHandle:** Passing a JsonObject that does not conform to the required structure will result in a runtime crash. Always use *shouldHandle* as a guard if you are not certain of the data's schema.

## Data Pipeline
This class is a key stage in the data deserialization pipeline for world generation. It transforms static data into an executable object.

> Flow:
> Raw JSON File -> Gson Parser -> JsonObject -> **NoiseHeightThresholdInterpreterJsonLoader** -> NoiseHeightThresholdInterpreter Object -> Procedural Generator Engine


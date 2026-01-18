---
description: Architectural reference for AbstractCellJitterJsonLoader
---

# AbstractCellJitterJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public abstract class AbstractCellJitterJsonLoader<K extends SeedResource, T> extends JsonLoader<K, T> {
```

## Architecture & Concepts
AbstractCellJitterJsonLoader is a foundational component within the procedural generation framework. It serves as a specialized, abstract base class for parsing positional randomization—or "jitter"—data from JSON configuration files.

Its primary architectural role is to enforce a standardized contract for how jitter values are defined and loaded. By extending this class, concrete loaders for specific world generation features (e.g., biome placement, object spawning) can inherit a consistent and robust mechanism for deserializing jitter configurations without duplicating logic. This class embodies the "Don't Repeat Yourself" (DRY) principle for a very common data structure in procedural systems.

It operates as a short-lived, stateful parser, instantiated to handle a single JSON element and discarded upon completion. It is a direct dependency for any system that needs to translate a JSON configuration into a runtime CellJitter object.

### Lifecycle & Ownership
- **Creation:** An instance of a *concrete subclass* is created by a higher-level manager within the procedural generation pipeline, such as a WorldGenerator or a BiomeLoader. It is passed the specific JsonElement from which to extract jitter data.
- **Scope:** The object's lifetime is strictly limited to the duration of the parsing operation for a single JSON configuration. It is a transient object, not a long-lived service.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the calling method has received the resulting CellJitter object and the loader instance goes out of scope. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
- **State:** The class is stateful for the duration of its lifecycle. It holds immutable references to the input SeedString, data folder Path, and the target JsonElement, all provided during construction. It does not cache results or modify its internal state after creation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within a larger data loading sequence. The underlying Gson JsonElement is also not designed for concurrent access.

**WARNING:** Confining instances of this loader and its subclasses to a single thread is critical to prevent data corruption and non-deterministic behavior in the procedural generation engine.

## API Surface
The public contract is exposed to subclasses via protected methods. The primary purpose is to provide a simple, high-level entry point for parsing a complete jitter configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadJitter() | CellJitter | O(1) | Orchestrates the loading of jitter values. Reads a default and axis-specific overrides from the JSON, returning a fully constructed CellJitter object. |
| loadDefaultJitter() | double | O(1) | Loads the global "Jitter" value, which serves as a fallback for any unspecified axis. |
| loadJitterX(default) | double | O(1) | Loads the "JitterX" value, or returns the provided default if not present. |
| loadJitterY(default) | double | O(1) | Loads the "JitterY" value, or returns the provided default if not present. |
| loadJitterZ(default) | double | O(1) | Loads the "JitterZ" value, or returns the provided default if not present. |

## Integration Patterns

### Standard Usage
This class is abstract and cannot be used directly. A concrete implementation consumes it to add jitter-loading capabilities. The typical pattern involves calling `loadJitter` from within the main `load` method of the subclass.

```java
// Example of a concrete subclass for loading a "Zone"
public class ZoneJsonLoader extends AbstractCellJitterJsonLoader<ZoneSeed, Zone> {

    public ZoneJsonLoader(ZoneSeed seed, Path dataFolder, JsonElement json) {
        super(seed, dataFolder, json);
    }

    @Override
    public Zone load() {
        // Use the inherited helper to parse the jitter block
        CellJitter placementJitter = this.loadJitter();

        // ... load other zone-specific properties from JSON ...
        String zoneName = this.get("name").getAsString();

        return new Zone(zoneName, placementJitter);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** You cannot instantiate this class using `new AbstractCellJitterJsonLoader()`. It must be extended.
- **State Re-use:** Do not attempt to re-use a loader instance to parse a different JsonElement. Each parsing operation requires a new instance.
- **Concurrent Access:** Never pass a loader instance to another thread or access it concurrently. The behavior is undefined and will lead to unstable world generation.

## Data Pipeline
This class functions as a deserialization step in the procedural content pipeline. It transforms structured text data into a specific in-memory object used by the generation algorithms.

> Flow:
> JSON File on Disk -> Gson Parser -> JsonElement -> **Concrete Subclass of AbstractCellJitterJsonLoader** -> `loadJitter()` call -> CellJitter Object -> Procedural Placement Algorithm


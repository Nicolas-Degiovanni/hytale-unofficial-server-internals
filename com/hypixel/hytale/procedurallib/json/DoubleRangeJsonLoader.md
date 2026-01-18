---
description: Architectural reference for DoubleRangeJsonLoader
---

# DoubleRangeJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient Utility

## Definition
```java
// Signature
public class DoubleRangeJsonLoader<K extends SeedResource> extends JsonLoader<K, IDoubleRange> {
```

## Architecture & Concepts

The DoubleRangeJsonLoader is a specialized factory component responsible for parsing a Gson JsonElement and transforming it into a concrete IDoubleRange object. Its primary role is to act as a bridge between declarative data configurations (JSON) and the imperative logic of the procedural generation engine.

This loader is designed for high flexibility, capable of interpreting multiple JSON structures to define a numeric range:
1.  **Constant:** A single JSON number (e.g., `5.0`).
2.  **Uniform Range:** A two-element JSON array (e.g., `[0.0, 1.0]`) or a JSON object with *Min* and *Max* keys.
3.  **Threshold-Based Range:** A complex JSON object with *Thresholds* and *Values* arrays, allowing for non-uniform, stepped value mapping.

A key architectural feature is the injectable DoubleToDoubleFunction. This allows the loader to perform a final transformation or scaling pass on all parsed numeric values before constructing the final IDoubleRange object. This decouples raw data values from their in-engine representation, enabling adjustments without modifying source JSON files.

## Lifecycle & Ownership

-   **Creation:** Instantiated on-demand by a higher-level configuration parser or procedural system when it encounters a data block that requires an IDoubleRange. It is **not** a managed service or singleton.
-   **Scope:** Extremely short-lived. An instance's scope is typically confined to the single method call responsible for parsing one specific JSON element.
-   **Destruction:** The object is intended to be immediately eligible for garbage collection after the `load` method has been called and its result has been retrieved. There is no cleanup logic or explicit destruction required.

## Internal State & Concurrency

-   **State:** The object is effectively **immutable** after construction. All internal fields, including default values and the transformation function, are finalized during instantiation. The `load` method is a pure function with respect to the object's state; it reads configuration but does not modify it.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state, a single instance can be used across multiple threads without risk of data corruption. However, its design as a transient utility makes such a use case highly improbable. The class contains no internal locks.

## API Surface

The public API is dominated by its constructors for configuration and a single `load` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IDoubleRange | O(N) | Parses the configured JsonElement into an IDoubleRange. Throws IllegalStateException if the JSON is malformed or violates structural rules. Complexity is O(N) for threshold-based ranges where N is the number of thresholds, and O(1) for all other cases. |

## Integration Patterns

### Standard Usage

The loader is designed to be instantiated, used once, and discarded. The calling system is responsible for providing the raw JsonElement and any necessary default values or transformation logic.

```java
// Assume 'configElement' is a JsonElement from a larger config file.
// Assume 'contextSeed' is a valid SeedString for logging/debugging.

// Create a loader with default values of 0.0 to 1.0
DoubleRangeJsonLoader<ZoneSeed> loader = new DoubleRangeJsonLoader<>(
    contextSeed,
    dataFolderPath,
    configElement,
    0.0,
    1.0
);

// Execute the parsing and get the concrete object
IDoubleRange biomeHeight = loader.load();

// The procedural engine can now use the biomeHeight object
float height = biomeHeight.get(proceduralNoiseValue);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching:** Do not cache and reuse instances of DoubleRangeJsonLoader. They are lightweight and tied to a specific JsonElement from their constructor. Caching provides no performance benefit and creates brittle code.
-   **Ignoring Exceptions:** The `load` method throws unchecked exceptions on malformed data. This is by design. Failure to parse a critical configuration value should be treated as a fatal error for the current task. Do not wrap `load` calls in an empty catch block.
-   **External State Modification:** Do not attempt to modify the input JsonElement on another thread while the `load` method is executing. While the loader itself is thread-safe, the input data it operates on is not guaranteed to be.

## Data Pipeline

The DoubleRangeJsonLoader is a specific, well-defined stage in the procedural asset loading pipeline. It consumes raw JSON structures and produces a high-level, usable game logic object.

> Flow:
> JSON Configuration File → GSON Parser → JsonElement → **DoubleRangeJsonLoader** → IDoubleRange Object → Procedural Generation Algorithm


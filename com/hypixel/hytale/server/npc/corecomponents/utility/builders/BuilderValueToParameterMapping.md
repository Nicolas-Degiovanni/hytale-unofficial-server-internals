---
description: Architectural reference for BuilderValueToParameterMapping
---

# BuilderValueToParameterMapping

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderValueToParameterMapping extends BuilderBase<BuilderValueToParameterMapping.ValueToParameterMapping> {
```

## Architecture & Concepts
The BuilderValueToParameterMapping class is a foundational component within the server's data-driven NPC configuration system. It functions as a transient factory, responsible for deserializing a mapping rule from a data source (typically JSON) and producing an optimized, runtime-ready mapping object.

Its primary architectural purpose is to decouple an NPC's internal state from its behavior parameters. This allows designers to define complex relationships in data files—for example, linking a value from an NPC's ValueStore (e.g., "current_health_percentage") to a behavior tree parameter (e.g., "flee_priority") without requiring code changes.

The core concept is **load-time name resolution**. During the asset loading phase, this builder resolves human-readable string identifiers for values and parameters into high-performance integer-based slot indices. The final product, a ValueToParameterMapping object, contains only these integer slots, enabling extremely fast lookups during the game loop. This pattern is critical for performance, avoiding repeated string comparisons or hash map lookups at runtime.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's asset loading and deserialization framework when it encounters a corresponding mapping definition within an NPC asset file.
-   **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single mapping rule. Its lifecycle is confined to the sequential calls of readConfig and build.
-   **Destruction:** The builder object is immediately eligible for garbage collection after the build method returns its product, the ValueToParameterMapping instance. The builder itself is never retained by any system. The produced ValueToParameterMapping object is then owned by the NPC behavior component that required it.

## Internal State & Concurrency
-   **State:** The builder's state is highly mutable but transient. Fields like type, fromValue, and toParameter are populated sequentially during the call to readConfig. This state is temporary scaffolding used solely to construct the final, effectively immutable ValueToParameterMapping object.
-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed for single-threaded execution within the asset loading pipeline. Concurrent calls to readConfig or build on the same instance will result in a corrupted and unpredictable state. This design is intentional, as asset parsing is typically a synchronous, single-threaded operation for a given asset.

## API Surface
The public API is designed for use by the asset deserialization framework, not for general game logic development.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder | O(k) | Populates the builder's internal state from a JSON structure. k is the number of keys in the JSON object. Throws exceptions on malformed or missing data. |
| build(BuilderSupport support) | ValueToParameterMapping | O(1) | Constructs the final, optimized mapping object. Resolves string names to integer slots using the provided BuilderSupport context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked transparently by the NPC asset loading system. The following conceptual example illustrates its place in the pipeline.

```java
// Conceptual example within an asset loader
JsonElement mappingConfig = parseNpcAssetFile("...").get("parameterMappings");

// The framework would instantiate and use the builder like this:
BuilderValueToParameterMapping builder = new BuilderValueToParameterMapping();
builder.readConfig(mappingConfig);
ValueToParameterMapping runtimeMapping = builder.build(assetLoadingContext.getBuilderSupport());

// The 'runtimeMapping' object is then passed to an NPC behavior component.
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Invocation:** Never create or use this builder during the main game loop. It is strictly a load-time component. Its dependencies and performance characteristics are not suited for runtime execution.
-   **State Reuse:** Do not attempt to reuse a single builder instance to process multiple mapping configurations. Each definition from the source data requires a new, clean instance of the builder.
-   **Manual Construction:** Avoid instantiating this class with `new`. It provides no value without being driven by the asset loading framework that calls `readConfig` and provides a valid `BuilderSupport` context to `build`.

## Data Pipeline
This component acts as a transformation step in the data pipeline that converts human-readable configuration into a performance-optimized runtime object.

> Flow:
> NPC Asset File (JSON) -> JSON Parser -> **BuilderValueToParameterMapping** -> ValueToParameterMapping (Optimized Product) -> NPC Behavior Component State


---
description: Architectural reference for ClimateRuleJsonLoader
---

# ClimateRuleJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateRuleJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateSearch.Rule> {
```

## Architecture & Concepts
The ClimateRuleJsonLoader is a specialized, single-purpose data loader within the server-side world generation subsystem. Its sole responsibility is to translate a specific JSON data structure into a strongly-typed, in-memory ClimateSearch.Rule object.

This class acts as a deserialization and validation bridge between raw configuration files and the core world generation engine. By extending the generic JsonLoader, it adheres to a standardized pattern for resource loading, ensuring consistency across the procedural library. It encapsulates the specific schema for a climate rule—which defines target values and weights for parameters like continentality, temperature, and humidity—abstracting the parsing logic away from the primary generation algorithms.

In the broader system, this loader is a critical component for enabling data-driven world design. It allows designers to define complex biome placement rules in simple JSON files without modifying core engine code.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration service or asset loader. It is instantiated with the raw JsonElement it is expected to parse, along with contextual information like the data folder path. It is not managed by a dependency injection container or service registry.

- **Scope:** The object's lifetime is exceptionally short and is strictly confined to the scope of the method that invokes it. It is a classic transient utility object, designed to perform one operation and then be discarded.

- **Destruction:** The instance becomes eligible for garbage collection immediately after the `load` method returns the ClimateSearch.Rule object. It holds no persistent state and is not referenced by any long-lived services.

## Internal State & Concurrency
- **State:** The loader's state consists of the JsonElement and Path provided at construction. This state is treated as immutable for the object's lifetime; it is read from but never modified.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used synchronously within a single thread, typically the main world generation thread. Concurrent calls to `load` on the same instance or access to its internal JSON state will result in undefined behavior and potential data corruption.

## API Surface
The public contract is minimal, focused entirely on the loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateSearch.Rule | O(1) | Parses the internal JSON data into a ClimateSearch.Rule. Throws a JsonSyntaxException or similar runtime exception if required fields are missing or malformed. |

## Integration Patterns

### Standard Usage
The loader is intended to be used in a "create, use, discard" pattern. A managing service reads a configuration file, extracts the relevant JSON section, and uses the loader to perform the transformation.

```java
// Assume 'climateRuleJson' is a JsonElement from a larger config file
// and 'seed' is a valid SeedString instance.
ClimateRuleJsonLoader loader = new ClimateRuleJsonLoader(seed, dataFolderPath, climateRuleJson);
ClimateSearch.Rule rule = loader.load();

if (rule != null) {
    // The resulting 'rule' object is now used by the climate simulation engine.
    climateSystem.addRule(rule);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse instances of ClimateRuleJsonLoader. Each distinct JSON object to be parsed requires a new loader instance.

- **Concurrent Access:** Never pass a loader instance to another thread or access it from multiple threads simultaneously. All loading operations must be synchronized by the calling system.

- **External Modification:** Do not attempt to modify the internal JsonElement of the loader after it has been constructed. The loader expects its source data to be static.

## Data Pipeline
This class is a single, critical step in the data flow from disk configuration to the live world generator. It transforms declarative data into an executable rule.

> Flow:
> World Configuration File (JSON) -> Asset Loading Service -> **ClimateRuleJsonLoader** -> ClimateSearch.Rule (In-Memory Object) -> Climate Search Algorithm -> Biome Placement Engine


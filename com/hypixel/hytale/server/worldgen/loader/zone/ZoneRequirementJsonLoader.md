---
description: Architectural reference for ZoneRequirementJsonLoader
---

# ZoneRequirementJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZoneRequirementJsonLoader extends JsonLoader<SeedStringResource, Set<String>> {
```

## Architecture & Concepts
The ZoneRequirementJsonLoader is a specialized, short-lived parser within the server-side world generation framework. Its primary function is to interpret a specific `JsonElement` that defines the set of all zones required for a procedural generation task.

Architecturally, this class functions as an **Orchestrator** and **Aggregator**. It does not perform low-level parsing itself. Instead, it delegates the processing of distinct JSON sections—namely `MaskMapping` and `UniqueZones`—to other, more specialized loaders (ZoneColorMappingJsonLoader and UniqueZoneEntryJsonLoader). It then aggregates the results from these subordinate loaders into a single, unified `Set<String>` of zone identifiers.

This component is a key part of Hytale's data-driven world generation design, enabling developers to define complex zone relationships and requirements in external configuration files rather than hardcoding them into the engine.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level configuration service or world generator when it needs to resolve a set of required zones from a JSON definition. It is not a managed singleton.
- **Scope:** The object's lifetime is extremely brief, typically confined to the execution scope of a single method call. It is created, used to call `load`, and then immediately becomes eligible for garbage collection.
- **Destruction:** Handled automatically by the Java Garbage Collector once the `load` method returns and no more references to the instance exist.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The context required for its operation (seed, data path, and the JSON element) is provided via its constructor and is treated as immutable during the `load` operation. It performs no caching and retains no data after the `load` method completes.
- **Thread Safety:** This class is **not thread-safe** and is designed for synchronous, single-threaded execution. An instance should never be shared across threads. The primary risk of concurrent use would be unpredictable behavior caused by potential race conditions on the underlying `JsonElement` if it were modified externally during a `load` operation.

## API Surface
The public contract is minimal, consisting of the constructor for instantiation and the `load` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | Set&lt;String&gt; | O(N+M) | Parses the configured JSON, delegating to sub-loaders and aggregating a complete set of zone names. Throws `IllegalArgumentException` if the critical `MaskMapping` key is absent in the source JSON. |

## Integration Patterns

### Standard Usage
The loader is intended to be used as part of a larger data loading sequence. A parent system parses a configuration file, extracts the relevant JSON fragment, and passes it to this loader to resolve the zone identifiers.

```java
// Conceptual example of a parent service using the loader
JsonElement zoneRequirementsJson = rootConfig.get("zoneRequirements");
SeedString<SeedStringResource> currentSeed = worldGenContext.getSeed();
Path dataPath = worldGenContext.getDataPath();

// Instantiate, use, and discard
ZoneRequirementJsonLoader loader = new ZoneRequirementJsonLoader(currentSeed, dataPath, zoneRequirementsJson);
Set<String> requiredZones = loader.load();

// requiredZones is now ready for use by the world generator
worldGenerator.processRequiredZones(requiredZones);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not retain and reuse instances of ZoneRequirementJsonLoader. They are cheap to create and are not designed to be long-lived. Create a new instance for each distinct loading operation.
- **Ignoring Exceptions:** The `IllegalArgumentException` thrown on a missing `MaskMapping` indicates a corrupt or invalid configuration file. This is a critical error that must be caught and handled to prevent the world generator from operating with incomplete data.
- **External Modification:** Modifying the input `JsonElement` from another thread while `load` is executing is strictly forbidden and will lead to undefined behavior.

## Data Pipeline
This loader acts as a transformation step, converting a structured JSON object into a simple, flat set of strings for consumption by downstream world generation systems.

> Flow:
> WorldGen Config File -> Root JSON Parser -> `JsonElement` (for zone requirements) -> **ZoneRequirementJsonLoader** -> `Set<String>` (Zone Identifiers) -> World Generation Logic


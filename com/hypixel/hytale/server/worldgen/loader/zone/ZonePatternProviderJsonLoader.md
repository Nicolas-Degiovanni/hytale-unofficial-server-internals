---
description: Architectural reference for ZonePatternProviderJsonLoader
---

# ZonePatternProviderJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZonePatternProviderJsonLoader extends JsonLoader<SeedStringResource, ZonePatternProvider> {
```

## Architecture & Concepts
The ZonePatternProviderJsonLoader is a factory class responsible for constructing a ZonePatternProvider from a JSON configuration block. It is a critical component in the server-side world generation pipeline, acting as the bridge between raw configuration data and a functional, in-memory representation of how world zones are spatially distributed.

This loader does not perform all parsing logic itself. Instead, it functions as an orchestrator, delegating the parsing of specific JSON sub-sections to more specialized loaders, such as PointGeneratorJsonLoader and ZoneColorMappingJsonLoader. This compositional approach adheres to the Single Responsibility Principle, keeping the loader focused on assembly and validation.

Its primary architectural role is to interpret how a given MaskProvider—a procedural or bitmap-based mask defining broad terrain features—maps to specific gameplay Zones. It performs critical integrity checks to ensure that every color or value in the mask corresponds to a defined zone, preventing catastrophic failures during the chunk generation phase.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level world generation service during the initial loading and parsing of worldgen configuration files. It is provided with a specific JsonElement from the configuration, a pre-initialized MaskProvider, and a world-generation seed.
- **Scope:** This object is short-lived and stateful. Its existence is scoped entirely to the configuration loading process. It is designed for a single, one-shot invocation of its `load` method.
- **Destruction:** The loader becomes eligible for garbage collection immediately after the fully constructed ZonePatternProvider is returned from the `load` method. There are no external references to it after this point.

## Internal State & Concurrency
- **State:** The ZonePatternProviderJsonLoader holds mutable state, including the MaskProvider, the array of available Zones, and a lookup map for those zones. This state is configured post-construction via the `setZones` method. The state is considered stable only after `setZones` has been called and before `load` is invoked.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated, configured, and used by a single thread during the server's bootstrap sequence. Concurrent calls to `setZones` or `load` will result in unpredictable behavior and race conditions. Do not share instances of this loader across threads.

## API Surface
The public API is minimal, reflecting its role as a single-purpose factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setZones(Zone[] zones) | void | O(N) | Populates the loader with the set of all available zones. **Must be called before `load`**. |
| load() | ZonePatternProvider | O(Complex) | Orchestrates the loading and validation process, returning a fully initialized ZonePatternProvider. |
| loadZoneRequirement() | Set<String> | O(Complex) | Delegates to ZoneRequirementJsonLoader to parse the set of required zone names from the JSON. |

## Integration Patterns

### Standard Usage
The loader follows a strict initialize-then-load pattern. It is intended to be used by a central world generation configuration service.

```java
// 1. Obtain the loader instance from a higher-level service (not shown)
//    which provides the seed, dataFolder, json, and maskProvider.
ZonePatternProviderJsonLoader loader = new ZonePatternProviderJsonLoader(seed, path, json, mask);

// 2. Provide the complete set of defined zones.
//    This is a critical dependency for mapping.
Zone[] allAvailableZones = zoneRegistry.getAllZones();
loader.setZones(allAvailableZones);

// 3. Execute the load operation to get the final provider.
ZonePatternProvider provider = loader.load();

// The loader instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Calling `load` before `setZones`:** Invoking `load` without first calling `setZones` will result in a NullPointerException, as the internal zone lookup map will be uninitialized.
- **Reusing Instances:** This loader is stateful and not designed for reuse. Do not hold a reference to it and call `load` multiple times. Create a new instance for each configuration block.
- **Concurrent Access:** Accessing a single loader instance from multiple threads is unsafe and will lead to data corruption or runtime exceptions.

## Data Pipeline
The ZonePatternProviderJsonLoader transforms configuration data into a functional service object. It sits between the raw file system assets and the active world generator.

> Flow:
> WorldGen JSON Config -> Parent Config Parser -> **ZonePatternProviderJsonLoader** -> ZonePatternProvider -> World Generator Service


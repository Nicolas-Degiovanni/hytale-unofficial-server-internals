---
description: Architectural reference for ZoneColorMappingJsonLoader
---

# ZoneColorMappingJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZoneColorMappingJsonLoader extends JsonLoader<SeedStringResource, ZoneColorMapping> {
```

## Architecture & Concepts
The ZoneColorMappingJsonLoader is a specialized, short-lived parser responsible for translating a JSON-based color-to-zone mapping into a runtime ZoneColorMapping object. It acts as a critical bridge during the world generation bootstrap process, linking abstract configuration data (JSON) with concrete, in-memory game objects (Zones).

Its primary architectural role is that of a **resolver** and **linker**. It does not discover or create Zone definitions itself. Instead, it depends on a pre-populated registry of all available Zones, provided during its construction. Its function is to parse the color map, look up each zone name in the provided registry, and build the final, memory-optimized mapping object used by the world generator to paint biomes based on a source image or map.

The inclusion of the static utility method, collectZones, strongly implies a two-pass loading strategy is expected by the parent system.
1.  **Pass 1 (Discovery):** The system uses collectZones to scan the JSON configuration and aggregate a set of all required Zone names.
2.  **Pass 2 (Resolution):** After loading the required Zones into a registry (a Map), the system instantiates this loader with that registry to perform the final linking.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation orchestrator, typically after parsing the raw JSON configuration file and after the primary Zone registry has been fully populated. It is not a managed service or singleton.
-   **Scope:** Extremely short-lived. An instance of this class is intended to exist only for the duration of a single `load` operation. Its purpose is fulfilled the moment it returns the ZoneColorMapping object.
-   **Destruction:** The object holds no persistent resources and manages no external state. It becomes eligible for garbage collection immediately after its `load` method completes.

## Internal State & Concurrency
-   **State:** The loader's state is provided entirely through its constructor (the JSON data and the zone lookup map) and is treated as immutable for the object's lifetime. The `load` method is a pure transformation of this initial state into a new object.
-   **Thread Safety:** This class is **not thread-safe** and is designed for synchronous execution. It must not be accessed from multiple threads. The `zoneLookup` map it receives is not internally protected, and concurrent modification of that map during a `load` operation will lead to undefined behavior and likely a runtime crash. All world generation data loading should be performed in a single, controlled thread.

## API Surface
The public API is minimal, reflecting its focused, single-purpose design.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ZoneColorMapping | O(N) | Parses the JSON, resolves all zone names, and returns the final mapping object. Throws IllegalArgumentException if any zone name is not found in the provided lookup map. |
| collectZones(Set, JsonElement) | static void | O(N) | A static utility to pre-scan a JSON structure and populate a Set with all referenced zone names. This is used for dependency discovery before loading. |

## Integration Patterns

### Standard Usage
The correct usage pattern is a two-pass process orchestrated by a higher-level system. This ensures that all required Zone dependencies are loaded before attempting to link them.

```java
// Assume 'jsonConfig' is a JsonElement read from a file
// Assume 'zoneRegistry' is the service that loads Zone objects

// Pass 1: Discover all required zone names
Set<String> requiredZones = new HashSet<>();
ZoneColorMappingJsonLoader.collectZones(requiredZones, jsonConfig);

// Load the actual Zone objects based on the discovered names
Map<String, Zone> loadedZoneMap = zoneRegistry.loadZones(requiredZones);

// Pass 2: Instantiate the loader and resolve the mapping
ZoneColorMappingJsonLoader loader = new ZoneColorMappingJsonLoader(seed, dataPath, jsonConfig, loadedZoneMap);
ZoneColorMapping finalMapping = loader.load();

// The 'loader' instance is no longer needed and can be discarded
```

### Anti-Patterns (Do NOT do this)
-   **Incomplete Registry:** Do not instantiate this loader with an incomplete `zoneLookup` map. If the JSON references a zone name that is not present in the map, the `load` method will fail with a hard crash (IllegalArgumentException). All dependencies must be satisfied beforehand.
-   **Instance Reuse:** Do not attempt to reuse a loader instance for multiple configurations. It is designed as a single-use, transient object.
-   **Concurrent Execution:** Do not call `load` from multiple threads or modify the `zoneLookup` map from another thread while `load` is executing.

## Data Pipeline
This loader is a transformation step in the world generation asset pipeline. It consumes configuration and a data registry to produce a runtime object.

> Flow:
> ZoneColorMap.json -> Json Parser -> `JsonElement` + `Map<String, Zone>` -> **ZoneColorMappingJsonLoader** -> `ZoneColorMapping` -> World Generator Engine


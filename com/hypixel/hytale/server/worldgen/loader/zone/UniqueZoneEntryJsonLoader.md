---
description: Architectural reference for UniqueZoneEntryJsonLoader
---

# UniqueZoneEntryJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.zone
**Type:** Transient

## Definition
```java
// Signature
public class UniqueZoneEntryJsonLoader extends JsonLoader<SeedStringResource, Zone.UniqueEntry[]> {
```

## Architecture & Concepts
The UniqueZoneEntryJsonLoader is a specialized, single-purpose data transformer within the server's world generation pipeline. It serves as the bridge between raw JSON configuration on disk and the engine's in-memory representation of unique, non-repeating world generation zones.

Its primary responsibility is to parse a JSON array defining unique zones and convert it into a strongly-typed `Zone.UniqueEntry[]` array. This process involves not only data conversion but also critical validation and dependency resolution. Each entry in the JSON specifies a zone by name, a unique color identifier for the world map, parent zone colors for placement rules, and placement parameters like radius and padding.

This loader is a dependent component; it cannot function in isolation. It relies on a pre-populated `zoneLookup` map, which contains all previously loaded Zone definitions. This design enforces a strict loading order: all standard zones must be loaded and indexed before unique zones can be processed and linked to them.

### Lifecycle & Ownership
-   **Creation:** An instance is created by a higher-level world generation orchestrator, typically the service responsible for parsing the main world generation configuration file. The orchestrator provides the specific JSON snippet for unique zones and the completed zone dependency map to the constructor.
-   **Scope:** The object's lifetime is extremely short and confined to a single method call. It is created for the sole purpose of executing the `load` method once.
-   **Destruction:** The instance is not explicitly managed or destroyed. It becomes eligible for garbage collection as soon as the `load` method returns and the caller's reference to it goes out of scope.

## Internal State & Concurrency
-   **State:** The loader's state is provided entirely at construction and is treated as immutable. It holds references to the source JSON, the data folder path, and the `zoneLookup` map. It does not cache any data or modify its internal state during the loading process.
-   **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. It is intended to be instantiated and used by a single thread as part of a sequential loading process. Sharing an instance across multiple threads is a severe anti-pattern and will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | Zone.UniqueEntry[] | O(N) | Parses the JSON array provided at construction into an array of Zone.UniqueEntry objects. Throws an Error on malformed data or if a zone name cannot be resolved. |
| collectZones(Set, JsonElement) | static void | O(N) | A static utility method to pre-scan a JSON element and populate a Set with all required zone names. This is a dependency-gathering step. |

## Integration Patterns

### Standard Usage
The loader is used as part of a multi-stage world generation setup process. The orchestrator first uses the static `collectZones` method to determine all zone dependencies, loads those zones, and only then instantiates the loader to perform the final transformation.

```java
// 1. A higher-level service parses the main worldgen config
JsonObject worldGenConfig = loadWorldGenFile();
JsonElement uniqueZonesJson = worldGenConfig.get("UniqueZones");

// 2. Pre-collect all zone name dependencies
Set<String> requiredZones = new HashSet<>();
UniqueZoneEntryJsonLoader.collectZones(requiredZones, uniqueZonesJson);

// 3. Load all required zones into a map (logic not shown)
Map<String, Zone> zoneLookup = loadRequiredZones(requiredZones);

// 4. Instantiate the loader with dependencies and execute
UniqueZoneEntryJsonLoader loader = new UniqueZoneEntryJsonLoader(seed, dataPath, uniqueZonesJson, zoneLookup);
Zone.UniqueEntry[] uniqueEntries = loader.load();

// 5. Pass the result to the world generator
worldGenerator.setUniqueZoneEntries(uniqueEntries);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Dependencies:** Do not call `new UniqueZoneEntryJsonLoader(...)` with an empty or incomplete `zoneLookup` map. This will result in a runtime `Error` as soon as an unknown zone name is encountered.
-   **Instance Re-use:** This object is a transient worker, not a persistent service. Do not hold a reference to it after calling `load`. It should be created, used once, and discarded.
-   **Ignoring Pre-computation:** Bypassing the `collectZones` static method and attempting to load unique zones before standard zones are fully loaded will break the world generation pipeline.

## Data Pipeline
The flow of data through this component is linear and unidirectional, transforming configuration text into a structured, engine-ready data type.

> Flow:
> WorldGen JSON File -> Primary JSON Parser -> `UniqueZones` JsonElement -> **UniqueZoneEntryJsonLoader** -> `Zone.UniqueEntry[]` Array -> World Generator Service


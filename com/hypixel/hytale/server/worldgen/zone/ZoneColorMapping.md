---
description: Architectural reference for ZoneColorMapping
---

# ZoneColorMapping

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ZoneColorMapping {
```

## Architecture & Concepts
The ZoneColorMapping class is a fundamental component of the server-side world generation pipeline. It functions as a high-performance lookup table that translates integer RGB color values into one or more corresponding game world Zones.

Its primary role is to decouple the abstract, visual representation of a world's layout (often defined in a source image like a PNG file) from the concrete game logic that generates terrain, biomes, and points of interest. World designers can create a "zone map" where each pixel's color corresponds to a specific type of zone. The world generator then reads this image pixel by pixel, uses an instance of ZoneColorMapping to look up the color, and retrieves the appropriate Zone definitions to apply at that coordinate.

This class utilizes the fastutil library's Int2ObjectOpenHashMap for memory efficiency and high-speed, primitive-based lookups, which is critical for the performance-sensitive task of world generation.

### Lifecycle & Ownership
-   **Creation:** An instance of ZoneColorMapping is typically created by a higher-level world generation service or factory at the beginning of a world generation process. It is not a managed service and is instantiated directly.
-   **Scope:** The object's lifetime is bound to the specific world generation session it was created for. It is populated once with the world's zone-to-color configuration and then used for the duration of that generation task.
-   **Destruction:** As a plain Java object with no external resources, it is automatically eligible for garbage collection once the world generator that owns it is destroyed and all references to it are released.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. The core data structure is a hash map that is populated via the public `add` methods. The object is designed to be built up during an initialization phase and then treated as a read-only dictionary.

-   **Thread Safety:** This class is **not thread-safe**. The underlying `Int2ObjectOpenHashMap` provides no guarantees for concurrent access.

    **WARNING:** Modifying the mapping via `add` from one thread while another thread performs lookups via `get` will result in unpredictable behavior, including potential `ConcurrentModificationException`, data corruption, or infinite loops. All population of the map must be completed in a single-threaded context before it is passed to any multi-threaded generation workers.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(int rgb, Zone zone) | void | O(1) | Convenience method to map an RGB color to a single Zone. |
| add(int rgb, Zone[] zones) | void | O(1) | Maps an RGB color to an array of Zones, overwriting any previous mapping for that color. |
| get(int rgb) | Zone[] | O(1) | Retrieves the Zone array for a given RGB color. Returns null if no mapping exists. |

## Integration Patterns

### Standard Usage
The canonical use case involves creating an instance, populating it with all necessary color-to-zone mappings, and then passing the fully configured object to the core world generation logic for read-only lookups.

```java
// 1. Initialization Phase (e.g., during server startup or world loading)
ZoneColorMapping zoneMap = new ZoneColorMapping();
zoneMap.add(0xFF0000, new Zone[]{FOREST_ZONE}); // Red pixels map to Forest
zoneMap.add(0x0000FF, new Zone[]{OCEAN_ZONE, DEEP_OCEAN_ZONE}); // Blue pixels map to Ocean variants

// 2. Generation Phase (inside a world generator)
// The 'zoneMap' instance is now treated as read-only.
int pixelColor = sourceZoneImage.getRGB(x, y);
Zone[] zonesForPixel = zoneMap.get(pixelColor);

if (zonesForPixel != null) {
    // Apply zone logic for this coordinate
    worldChunk.applyZones(x, y, zonesForPixel);
} else {
    // Handle default or unknown zone color
}
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call `add` after the mapping has been passed to worker threads for generation. The object must be effectively immutable after its initial setup.
-   **Ignoring Null Returns:** The `get` method returns null for unmapped colors. Code that consumes this method must include a null check to prevent `NullPointerException`.
-   **Re-population in Loops:** Do not create and populate a new ZoneColorMapping inside a generation loop. It is a configuration object that should be built once and reused for millions of lookups.

## Data Pipeline
ZoneColorMapping is a key transformation step in the procedural generation data flow. It converts low-level pixel data into high-level, semantic game objects.

> Flow:
> Zone Map Asset (e.g., PNG) -> Image Loader -> `int` RGB Pixel Value -> **ZoneColorMapping.get()** -> `Zone[]` Array -> World Generator Logic -> Final World Chunk Data


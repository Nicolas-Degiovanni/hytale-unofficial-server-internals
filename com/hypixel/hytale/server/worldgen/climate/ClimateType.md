---
description: Architectural reference for ClimateType
---

# ClimateType

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Data Structure / Value Object

## Definition
```java
// Signature
public class ClimateType {
```

## Architecture & Concepts

The ClimateType class is an immutable data structure that represents a single node in the server's hierarchical climate system. It is a fundamental building block for world generation, defining the properties of a specific biome or environmental region.

This class does not contain any logic for world generation itself. Instead, it serves as a configuration object, holding the definitional data for a climate, such as its name, associated colors, and its relationship to other climates. Instances of ClimateType are organized into a tree or graph structure, likely managed by a higher-level service like a ClimateGraph. The `children` field explicitly defines this hierarchical relationship.

The static integer constants (e.g., IS_ISLAND, IS_SHORE, IS_OCEAN) are bitmasks. The world generation engine combines the unique ID of a base ClimateType with these flags to produce a final integer ID for a specific point in the world. This composite ID efficiently encodes not only the base climate (e.g., Forest, Desert) but also its geographical context (e.g., Forest Shore, Desert Island). The static `color` method demonstrates this pattern, decoding the bitmasks from a given ID to retrieve the correct color value.

## Lifecycle & Ownership

-   **Creation:** ClimateType instances are not created dynamically during gameplay. They are instantiated once during server startup by a configuration loading system that parses world generation definition files (e.g., JSON assets). The resulting objects are assembled into a complete climate graph.
-   **Scope:** An instance of ClimateType persists for the entire lifecycle of the server. It is part of the static world definition and is never modified or discarded while the server is running.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down and its containing ClimateGraph is dereferenced. There is no explicit cleanup logic.

## Internal State & Concurrency

-   **State:** The ClimateType class is **deeply immutable**. All of its fields are declared as `final`. Once an instance is constructed, its state cannot be changed. This design makes it a reliable and predictable data source for the world generator.
-   **Thread Safety:** This class is **inherently thread-safe**. Due to its immutability, instances can be safely read and shared across multiple world generation worker threads without any need for locks, synchronization, or other concurrency controls. This is a critical feature for a high-performance, multi-threaded world generation pipeline.

## API Surface

The primary API consists of static utility methods for interpreting and navigating the climate graph.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| name(parent, type) | static String | O(1) | Composes a fully qualified hierarchical name for a given climate type. |
| walk(type, visitor) | static void | O(N) | Traverses a climate subtree starting from a given node. N is the number of nodes in the subtree. |
| color(id, climate) | static int | O(1) | **Critical Utility.** Decodes a composite climate ID into a final integer color value. It extracts bitmask flags to select the correct color variant (e.g., land, shore, ocean). |

## Integration Patterns

### Standard Usage

Developers should not interact with ClimateType instances directly. Instead, they should use a higher-level manager, such as a ClimateGraph, to query for climate information. The static methods on ClimateType are then used as utilities to process the data returned from that manager.

```java
// Example: Retrieving a color for a given world-space climate ID
int climateIdFromWorld = world.getClimateIdAt(x, z);
ClimateGraph graph = server.getWorldGen().getClimateGraph();

// Use the static utility to resolve the ID to a color
int finalColor = ClimateType.color(climateIdFromWorld, graph);
mapRenderer.setColor(x, z, finalColor);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ClimateType()` in game logic. Climate definitions are static data loaded from configuration files at startup. Creating rogue instances will lead to an inconsistent and broken world state.
-   **State Mutation:** Although the fields are `final`, the `points` and `children` arrays are technically mutable. Modifying the contents of these arrays after construction is a severe violation of the class's design contract and will cause unpredictable, non-deterministic behavior in the world generator.

## Data Pipeline

ClimateType acts as a static data source, not a processing stage in a dynamic pipeline. Its primary role is to provide definitional data to other systems. The most common flow involves resolving a world-space climate ID to a concrete value, such as a color for a map.

> **Flow: ID to Color Resolution**
>
> World Data (contains integer `climateId`) -> Map Rendering System -> **ClimateType.color(climateId, graph)** -> Integer Color Value -> Rendered Map Pixel


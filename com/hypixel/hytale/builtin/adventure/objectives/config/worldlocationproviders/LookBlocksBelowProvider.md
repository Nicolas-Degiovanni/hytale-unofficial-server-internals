---
description: Architectural reference for LookBlocksBelowProvider
---

# LookBlocksBelowProvider

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.worldlocationproviders
**Type:** Transient

## Definition
```java
// Signature
public class LookBlocksBelowProvider extends WorldLocationProvider {
```

## Architecture & Concepts
The LookBlocksBelowProvider is a concrete implementation of the *Strategy Pattern*, extending the abstract WorldLocationProvider. Its purpose is to define and execute a specific world-scanning algorithm: finding a valid location by searching vertically downwards for a consecutive sequence of blocks that match a given set of block tags.

This component is a fundamental building block of the server-side Adventure Mode and questing systems. It allows designers to create objectives that are dynamically located based on the procedural generation of the world, rather than being tied to static, pre-defined coordinates. For example, an objective might require finding a location with at least three consecutive coal ore blocks, and this provider encapsulates the logic to perform that search.

Configuration is entirely data-driven, defined by the static CODEC field. This allows instances of LookBlocksBelowProvider to be deserialized directly from asset files, making the system highly flexible and extensible without requiring new Java code for different search criteria.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the engine's Codec system during the deserialization of game assets, specifically adventure objective configurations. The static CODEC field dictates how the object is constructed from a data source. Manual instantiation via the constructor is possible but circumvents the intended data-driven design.
- **Scope:** The object's lifetime is bound to the parent configuration that defines it, such as a specific quest or objective. It is not a global service and does not persist beyond the lifetime of its owner.
- **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded or goes out of scope. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The object is **effectively immutable** after its creation. All configuration fields, such as count and maxRange, are set during the deserialization process. A key internal state is the `blockTagsIndexes` array, which is a performance-oriented cache. It is populated once in the `afterDecode` lifecycle hook by resolving the string-based `blockTags` against the global AssetRegistry. This pre-computation avoids repeated string-to-index lookups during world scanning.
- **Thread Safety:** This class is **conditionally thread-safe**. The primary method, runCondition, does not mutate the object's internal state. Therefore, a single instance can be safely shared and executed by multiple threads, provided that the `World` and `WorldChunk` objects passed to it guarantee thread-safe read operations. The provider itself contains no internal locks or synchronization primitives.

## API Surface
The public contract is minimal, focused on executing the world-scanning logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| runCondition(World world, Vector3i position) | Vector3i | O(N) | Executes the scan. N is the configured `maxRange`. Returns the coordinates of the final valid block in the sequence, or null if the condition is not met within the search bounds. |

## Integration Patterns

### Standard Usage
This provider is not typically invoked directly. It is configured within a larger system, such as an objective, and executed by the engine when that objective needs to resolve a world location.

```java
// Conceptual example of how the engine might use the provider
// A developer would typically define this provider in a JSON asset file, not in code.

// 1. Engine deserializes the provider from an asset file.
WorldLocationProvider provider = assetLoader.load("my_objective_location_provider.json");

// 2. The objective system invokes the provider to find a suitable location.
Vector3i startSearchPos = new Vector3i(100, 80, 200);
Vector3i objectiveLocation = provider.runCondition(server.getWorld(), startSearchPos);

if (objectiveLocation != null) {
    // Spawn quest marker or entity at the found location.
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Avoid using `new LookBlocksBelowProvider()`. This pattern is brittle as it bypasses the robust validation and lifecycle hooks (like `afterDecode`) defined in the CODEC. All configuration should be managed in data assets.
- **Post-Creation Mutation:** Do not attempt to modify the provider's public fields after it has been constructed. The internal state, particularly `blockTagsIndexes`, will become desynchronized, leading to unpredictable behavior.
- **Invalid Range Configuration:** Configuring a `minRange` greater than `maxRange` will result in an `IllegalArgumentException` during asset loading. The validation logic within the CODEC prevents such invalid configurations from being loaded.

## Data Pipeline
The flow of data begins with a designer's configuration and ends with a resolved world coordinate.

> Flow:
> Adventure Objective Asset File -> Engine Codec Deserializer -> **LookBlocksBelowProvider Instance** (Resolves `blockTags` into `blockTagsIndexes` via AssetRegistry) -> ObjectiveManager invokes `runCondition` -> Scans `WorldChunk` block data -> Returns `Vector3i` or `null` to ObjectiveManager


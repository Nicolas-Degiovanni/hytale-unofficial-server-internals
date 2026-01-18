---
description: Architectural reference for FluidPatternHelper
---

# FluidPatternHelper

**Package:** com.hypixel.hytale.builtin.buildertools.utils
**Type:** Utility

## Definition
```java
// Signature
public final class FluidPatternHelper {
```

## Architecture & Concepts
FluidPatternHelper is a stateless, static utility class designed to resolve an item or block identifier into its corresponding fluid properties. It serves as a dedicated query service that abstracts the complex and indirect relationship between an item asset and the fluid it may place in the world.

Its primary function is to traverse the asset graph, starting from an item's definition, through its interaction configurations, to ultimately identify a PlaceFluidInteraction. This allows other systems, particularly the builder tools, to determine if a given tool or block is associated with a fluid (e.g., a water bucket) and to retrieve that fluid's unique ID and maximum level without needing to understand the underlying asset structure.

This class centralizes a piece of critical but niche domain logic, ensuring that the process for identifying a fluid-placing item is consistent and decoupled from the systems that consume this information.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. The private constructor enforces a purely static access pattern, which is characteristic of a utility class.
- **Scope:** As a collection of static methods, FluidPatternHelper has a global, application-wide scope. Its methods are available as soon as the class is loaded by the JVM.
- **Destruction:** Not applicable. No instances are created, so no cleanup is required.

## Internal State & Concurrency
- **State:** FluidPatternHelper is entirely stateless. It contains no member variables, caches, or any other form of internal state. Each method call is a pure function whose output depends solely on its inputs and the global state of the static asset maps it queries.
- **Thread Safety:** The class is inherently thread-safe. Since it has no state to mutate, race conditions within the helper itself are impossible.

   **WARNING:** The thread safety of this class is entirely dependent on the thread safety of the underlying static asset maps (Item.getAssetMap(), Fluid.getAssetMap(), etc.). It is assumed that these global asset registries are safe for concurrent read operations after their initial population during engine startup. Any mutation to these maps while FluidPatternHelper is in use will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFluidInfo(String itemKey) | FluidInfo | O(k) | Resolves an item key to fluid information. Returns null if the item is not a fluid item or if any asset lookup fails. Complexity is O(k) where k is the number of interactions on the item. |
| isFluidItem(String itemKey) | boolean | O(k) | A convenience method to check if an item is associated with a fluid. Internally calls getFluidInfo. |
| getFluidInfoFromBlockType(String blockTypeKey) | FluidInfo | O(k) | Resolves a block type key to fluid information. This is an alias for getFluidInfo, indicating that item and block keys may be used interchangeably in this context. |

## Integration Patterns

### Standard Usage
The intended use is to query the helper with an item or block key to retrieve its fluid data before performing a world modification or validation.

```java
// How a developer should normally use this
String heldItemKey = player.getHeldItem().getKey();
FluidPatternHelper.FluidInfo fluidInfo = FluidPatternHelper.getFluidInfo(heldItemKey);

if (fluidInfo != null) {
    // This item places a fluid. Use its properties.
    world.placeFluid(targetPos, fluidInfo.fluidId(), fluidInfo.fluidLevel());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate this class. The private constructor makes this impossible, and the design relies on static access.
- **Repetitive Lookups:** Avoid calling getFluidInfo for the same item key repeatedly within a tight loop or a single frame. The method performs multiple asset map lookups and is not cached internally. If the fluid information for an item is needed frequently, the caller is responsible for caching the result.

## Data Pipeline
FluidPatternHelper does not participate in a streaming data pipeline; rather, it executes a synchronous query and lookup chain. The flow of data is a series of dereferences across the static asset system.

> Flow:
> String itemKey -> **Item.getAssetMap()** -> Item Asset -> **RootInteraction.getAssetMap()** -> RootInteraction Asset -> **Interaction.getAssetMap()** -> PlaceFluidInteraction Asset -> String fluidKey -> **Fluid.getAssetMap()** -> **FluidPatternHelper.FluidInfo**


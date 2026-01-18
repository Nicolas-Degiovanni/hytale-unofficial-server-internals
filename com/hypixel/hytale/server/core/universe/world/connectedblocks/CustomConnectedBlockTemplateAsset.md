---
description: Architectural reference for CustomConnectedBlockTemplateAsset
---

# CustomConnectedBlockTemplateAsset

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Configuration Asset

## Definition
```java
// Signature
public class CustomConnectedBlockTemplateAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, CustomConnectedBlockTemplateAsset>> {
```

## Architecture & Concepts
The **CustomConnectedBlockTemplateAsset** is a data-driven configuration object that defines the rules for how a block's appearance changes based on its neighbors. It serves as the core component of the "connected blocks" system, which handles blocks like walls, fences, and glass panes that must adapt their geometry to form continuous structures.

Architecturally, this class acts as a passive rule set, not an active service. It is deserialized from JSON asset files at server startup and loaded into a central **AssetStore**. This design decouples the complex visual logic of block connectivity from the core world simulation loop. Instead of hard-coding connection rules in Java, game designers can define new connected block behaviors entirely through data files, which are then interpreted by this class.

The primary responsibility of this asset is to evaluate the state of a block's neighbors within the **World** and, based on a series of predefined patterns, determine the correct block model or "shape" to display.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via a constructor. They are instantiated and populated exclusively by the **AssetStore** framework during the server's asset loading phase. The static **CODEC** field defines the deserialization contract from the underlying JSON source.
-   **Scope:** An instance of this class is application-scoped. Once loaded from an asset file, it persists for the entire lifetime of the server session. The internal state should be considered immutable after this initial load.
-   **Destruction:** The asset and its associated data are garbage collected upon server shutdown when the **AssetRegistry** is cleared.

## Internal State & Concurrency
-   **State:** The internal state is mutable only during its initial creation by the asset loading system. After hydration, it becomes a read-only data container. It holds a map of **ConnectedBlockShape** objects, which in turn contain the matching patterns. It does not cache runtime data; each evaluation is performed against the live world state.
-   **Thread Safety:** This class is thread-safe for all read operations. Because its state is effectively immutable post-initialization, a single instance can be safely accessed by multiple threads simultaneously, such as parallel chunk generation or lighting update threads. The primary evaluation method, **getConnectedBlockType**, is a pure function with respect to the object's state and operates on a thread-local random number generator for default shape selection.

    **Warning:** While the asset itself is thread-safe, the **World** object passed into its methods must be managed with appropriate concurrency controls by the caller.

## API Surface
The public contract is minimal, centered around a single evaluation method and static accessors for the asset management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **Static**. Retrieves the central registry for all assets of this type. |
| getAssetMap() | DefaultAssetMap | O(1) | **Static**. Retrieves the map of all loaded assets, keyed by their ID. |
| getConnectedBlockType(...) | Optional | O(N\*M) | Evaluates all shapes (N) and their patterns (M) against the world state to find a matching block variant. Returns a result containing the new block type key and rotation, or empty if no pattern matches. |
| isDontUpdateAfterInitialPlacement() | boolean | O(1) | Returns a configuration flag indicating if the block should resist updates after its initial placement. |

## Integration Patterns

### Standard Usage
This asset must always be retrieved from the central **AssetStore**. It is typically used by world systems responsible for block updates or placement logic.

```java
// Correctly retrieve the asset from the central store
DefaultAssetMap<String, CustomConnectedBlockTemplateAsset> assetMap = CustomConnectedBlockTemplateAsset.getAssetMap();
CustomConnectedBlockTemplateAsset template = assetMap.get("hytale:stone_wall_template");

if (template != null) {
    // Evaluate the block's surroundings to get the correct variant
    Optional<ConnectedBlocksUtil.ConnectedBlockResult> result = template.getConnectedBlockType(
        world,
        coordinate,
        ruleSet,
        blockType,
        rotation,
        placementNormal,
        true,
        true
    );

    // Apply the result to the world
    result.ifPresent(r -> world.setBlock(coordinate, r.blockTypeKey(), r.rotation()));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new CustomConnectedBlockTemplateAsset()**. This will create an uninitialized and non-functional object that is not tracked by the asset system. It will fail at runtime.
-   **State Mutation:** Do not attempt to modify the internal state of a retrieved asset via reflection or other means. The system relies on these objects being immutable after they are loaded.
-   **Ignoring Context:** Calling **getConnectedBlockType** with stale **World** data will produce incorrect results. The method must be invoked with the most current representation of the block's neighborhood.

## Data Pipeline
The **CustomConnectedBlockTemplateAsset** is a critical step in the block update and rendering pipeline. It translates a high-level game event into a specific, low-level block state change.

> Flow:
> Game Event (e.g., Player places a block) -> World Update System -> AssetStore lookup for the relevant **CustomConnectedBlockTemplateAsset** -> **getConnectedBlockType** method evaluates neighboring blocks -> A **ConnectedBlockResult** is returned -> World system updates the block state at the coordinate -> The change is serialized and sent to clients.


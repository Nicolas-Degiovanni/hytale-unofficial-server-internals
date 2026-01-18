---
description: Architectural reference for RequiredBlockFaceSupport
---

# RequiredBlockFaceSupport

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Transient Data Object

## Definition
```java
// Signature
public class RequiredBlockFaceSupport implements NetworkSerializable<com.hypixel.hytale.protocol.RequiredBlockFaceSupport> {
```

## Architecture & Concepts
RequiredBlockFaceSupport is a data-driven configuration object that defines the rules for how a block face must be supported by adjacent blocks. It is a fundamental component of the world simulation engine, governing block placement, structural integrity, and complex multi-block arrangements.

This class is not a service or manager; it is a plain data structure that encapsulates a set of constraints. These constraints are defined by content creators in block asset files (e.g., JSON) and are deserialized into this object representation at server startup.

The primary architectural role of this class is to translate static, human-readable asset definitions into a highly optimized, in-memory format for the runtime engine. It forms the bridge between the Asset System and the Block Placement Validator. By using string identifiers like BlockSetId in assets and resolving them to integer indices during the loading process, it enables the world engine to perform validation checks with maximum performance.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during asset loading. The static CODEC field defines the deserialization logic from a data source into a Java object. Manual instantiation is strongly discouraged as it bypasses critical post-processing steps.
- **Scope:** An instance of RequiredBlockFaceSupport persists for the entire server session. It is owned by the configuration of a specific BlockType asset and is considered immutable after the initial loading phase is complete.
- **Destruction:** The object is garbage collected when the server shuts down and all asset registries are cleared.

## Internal State & Concurrency
- **State:** The object is **effectively immutable** after its creation and the execution of the *afterDecode* hook. The *afterDecode* process populates performance-critical fields like blockSetIndex and tagIndex by resolving their corresponding string IDs. This pre-calculation is a key optimization. The static *rotate* method reinforces the immutability pattern by returning a new, modified instance rather than altering the original.
- **Thread Safety:** **Thread-safe for reads.** Due to its immutable nature post-initialization, this object can be safely accessed by multiple concurrent systems (e.g., world generation threads, block update logic) without requiring any synchronization or locks.

**WARNING:** Modifying the state of a RequiredBlockFaceSupport object after it has been registered in the asset system will lead to severe, difficult-to-debug concurrency issues and inconsistent world simulation behavior.

## API Surface
The public API is designed for read-only access to configuration data and for transformation into other formats.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| rotate(original, yaw, pitch, roll) | static RequiredBlockFaceSupport | O(N) | Creates a new instance with rotated filler coordinates. N is the size of the filler array. |
| toPacket() | com.hypixel.hytale.protocol.RequiredBlockFaceSupport | O(N) | Transforms this server-side object into a network-ready packet. Throws IllegalArgumentException if an asset ID cannot be resolved to an index. |
| isAppliedToFiller(vector) | boolean | O(N) | Checks if this support rule applies to a specific sub-block position (filler). Returns true if no filler is defined. |

## Integration Patterns

### Standard Usage
This object is not typically used directly by game logic developers. Instead, it is consumed by the core world engine to validate block placements. The engine retrieves the relevant rule from a BlockType's asset definition and checks it against the state of neighboring blocks.

```java
// Hypothetical usage within a world validation system
BlockType torchType = AssetRegistry.getBlockType("hytale:torch");
RequiredBlockFaceSupport supportRule = torchType.getPlacementRules().getSupportForFace(BlockFace.SIDE);

// The engine checks if the adjacent block provides the required support type
boolean canPlace = world.getSupportSystem().checkSupport(adjacentBlockPos, supportRule);

if (!canPlace) {
    // Deny block placement
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RequiredBlockFaceSupport()`. The object must be created via the asset pipeline to ensure the *afterDecode* logic is executed, which correctly populates the integer index fields. Failure to do so will result in runtime exceptions or incorrect behavior.
- **State Mutation:** Do not modify the fields of this object after it has been loaded. For transformations, such as rotation, always use the provided static factory methods like *rotate* which produce a new instance.
- **Runtime ID Lookups:** Do not rely on the string-based IDs (e.g., getBlockSetId) for performance-critical runtime checks. Always use the pre-calculated integer indices (e.g., getBlockSetIndex) for fast, direct lookups.

## Data Pipeline
The RequiredBlockFaceSupport object is a key stage in the pipeline that transforms static asset data into network packets for the client.

> Flow:
> Block Asset File (JSON) -> Codec Deserialization -> **RequiredBlockFaceSupport** (Server-side, with resolved indices) -> toPacket() -> Network Packet (Client-side representation) -> Client Renderer

---


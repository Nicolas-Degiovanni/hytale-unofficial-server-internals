---
description: Architectural reference for BlockTypeTextures
---

# BlockTypeTextures

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class BlockTypeTextures {
```

## Architecture & Concepts

BlockTypeTextures is a server-side data model that represents the texture configuration for a single variant of a block. Its primary role is to serve as a deserialization target for on-disk asset definitions, typically written in a key-value format like JSON. It is a critical component of the asset pipeline, translating human-readable configuration into a network-optimized format for the client.

The core of this class is the static **CODEC** field. This `BuilderCodec` defines a flexible schema for content creators, enabling them to define textures with convenient shortcuts:
*   **All:** Assigns a single texture to all six faces of the block.
*   **Sides:** Assigns a texture to the four vertical faces (North, South, East, West).
*   **UpDown:** Assigns a texture to the top and bottom faces.

This class also supports an inheritance model, managed by the `appendInherited` calls within the CODEC definition. This allows a block variant to inherit texture definitions from a parent asset and override specific faces, reducing configuration duplication.

The **weight** property is fundamental for creating natural-looking terrain. When a block type has multiple texture variants, the engine uses their weights to perform a weighted random selection for each block placed in the world. This mechanism is essential for breaking up repetitive patterns in large surfaces like grass, dirt, and stone.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the server's asset loading system during the bootstrap phase. The static **CODEC** is invoked to parse a block's asset file and construct a corresponding BlockTypeTextures object. Direct, manual instantiation is an anti-pattern and circumvents the asset pipeline.
-   **Scope:** An instance of BlockTypeTextures has its lifetime scoped to its parent BlockType asset. It is loaded into memory once and persists as long as the BlockType is registered in the asset manager. For all practical purposes, it should be treated as an immutable object after initial loading.
-   **Destruction:** The object is eligible for garbage collection when the server's asset registry is reloaded or cleared, for instance during a server shutdown or a live asset hot-reload event.

## Internal State & Concurrency

-   **State:** The object's state is mutable *only* during the deserialization process controlled by the CODEC. Once fully constructed and loaded by the asset manager, its state is fixed. The fields representing texture paths default to a safe "Unknown.png" texture, preventing rendering errors for incomplete configurations.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be constructed and populated on a single thread within the asset loading system. Accessing an instance from multiple threads after it has been fully loaded is safe for reads, as its state is effectively immutable. Any external attempts to modify its state post-initialization will lead to unpredictable behavior and data desynchronization.

## API Surface

The public API is minimal, primarily exposing read-only access to the configured texture paths and providing a single transformation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUp() | String | O(1) | Returns the asset path for the top face texture. |
| getDown() | String | O(1) | Returns the asset path for the bottom face texture. |
| getNorth() | String | O(1) | Returns the asset path for the north face texture. |
| getSouth() | String | O(1) | Returns the asset path for the south face texture. |
| getEast() | String | O(1) | Returns the asset path for the east face texture. |
| getWest() | String | O(1) | Returns the asset path for the west face texture. |
| getWeight() | float | O(1) | Returns the raw selection weight for this texture variant. |
| toPacket(totalWeight) | BlockTextures | O(1) | Transforms this configuration object into a network packet. Normalizes its weight against the provided total. |

## Integration Patterns

### Standard Usage

This class is not meant to be used directly by most game logic. The engine's asset and networking layers handle its lifecycle. The primary integration point is the `toPacket` method, which is called to prepare block data for transmission to clients.

```java
// Hypothetical engine code to prepare a block type for the client
// This logic would exist deep within the server's asset management system.

// Assume 'variants' is a list of BlockTypeTextures loaded for a single block type.
float totalWeight = variants.stream().map(BlockTypeTextures::getWeight).reduce(0f, Float::sum);

// Transform each server-side config object into a network-ready packet.
List<BlockTextures> packets = variants.stream()
    .map(variant -> variant.toPacket(totalWeight))
    .collect(Collectors.toList());

// The 'packets' list would then be sent to the client.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create instances using `new BlockTypeTextures()`. The asset system relies on the **CODEC** to ensure proper initialization, validation, and inheritance. Bypassing it will result in an incomplete and invalid object.
-   **State Mutation:** Do not attempt to modify the state of a BlockTypeTextures object after it has been loaded. The asset system is the single source of truth; runtime modifications will be lost on asset reloads and will not be synchronized with clients.
-   **Incorrect Weight Calculation:** Do not use the raw value from `getWeight` for probability calculations. It must be normalized against the sum of all weights for a given block's variants. The `toPacket` method correctly handles this normalization for network transmission.

## Data Pipeline

BlockTypeTextures serves as a critical translation step between human-authored configuration and the binary network protocol.

> Flow:
> BlockType Asset File (JSON) -> Asset Deserializer (using **CODEC**) -> **BlockTypeTextures Instance** -> toPacket() -> BlockTextures (Network Packet) -> Client Renderer


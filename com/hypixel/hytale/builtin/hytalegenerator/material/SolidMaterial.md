---
description: Architectural reference for SolidMaterial
---

# SolidMaterial

**Package:** com.hypixel.hytale.builtin.hytalegenerator.material
**Type:** Value Object (Flyweight)

## Definition
```java
// Signature
public class SolidMaterial {
```

## Architecture & Concepts
The SolidMaterial class is an immutable data structure that represents the complete state of a single, non-air block within the world. It is a core component of the world generation system, designed for extreme memory efficiency through the **Flyweight** design pattern.

Instead of creating a new object for every block placed in the world, the engine reuses a single SolidMaterial instance for all blocks that share identical properties. For example, every standard, unrotated stone block throughout the entire world will reference the exact same SolidMaterial object in memory.

This object's identity is defined by its contents:
- **blockId:** The fundamental identifier for the type of block (e.g., stone, dirt, wood).
- **support, rotation, filler:** State parameters that define the block's orientation and variant.
- **holder:** An optional reference to a ChunkStore. This links the material to a specific chunk's storage, typically used for special blocks or tile entities that require context beyond their immediate type.

The creation and management of these objects are strictly controlled by the MaterialCache service, which acts as the central factory and repository.

### Lifecycle & Ownership
- **Creation:** SolidMaterial instances are created exclusively by the MaterialCache. The constructor is package-private to prevent direct instantiation from outside the material management system. A generator requests a material with specific properties, and the cache either returns a pre-existing, matching instance or constructs a new one.
- **Scope:** The object's lifetime is tied to the MaterialCache that created it. As long as the cache holds a reference, the object persists. Typically, this is for the entire server session.
- **Destruction:** Instances are eligible for garbage collection only when the MaterialCache is cleared and no other systems (like an in-progress world generation task) hold a reference.

## Internal State & Concurrency
- **State:** Strictly **immutable**. All fields are final and are assigned only once during construction. This design is critical for the Flyweight pattern to function correctly and safely.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, a SolidMaterial instance can be safely shared and read by multiple world generation threads simultaneously without any need for locks or other synchronization primitives.

## API Surface
The public contract of SolidMaterial is its data, exposed via public final fields. The most significant method is the static `contentHash`, which is central to the caching mechanism.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| contentHash(blockId, support, rotation, filler, holder) | static int | O(1) | Calculates the hash code for a potential material's state. Used by MaterialCache to perform lookups without creating a temporary object. |

## Integration Patterns

### Standard Usage
A developer should never instantiate SolidMaterial directly. Interaction is always mediated through a MaterialCache or a higher-level world generation context, which manages the caching.

```java
// Correct: Obtain a shared material instance from the cache
MaterialCache cache = context.getService(MaterialCache.class);
SolidMaterial stoneMaterial = cache.getSolidMaterial(STONE_BLOCK_ID, 0, 0, 0, null);

// Use the material to set a block in the world
world.setBlock(x, y, z, stoneMaterial);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create instances using reflection or by modifying package access. The system's memory efficiency relies on the central caching mechanism. Bypassing it will lead to severe performance degradation and memory bloat.
- **State Assumption:** Do not write code that expects to modify a SolidMaterial after it has been obtained. These objects are immutable and shared globally.

## Data Pipeline
SolidMaterial is not a processor; it is the data payload itself. It represents the outcome of a material selection decision within the world generation pipeline.

> Flow:
> World Generation Algorithm -> Decides Block Properties -> MaterialCache -> **SolidMaterial** (retrieved or created) -> ChunkStore Writer -> Block data written to chunk memory

---


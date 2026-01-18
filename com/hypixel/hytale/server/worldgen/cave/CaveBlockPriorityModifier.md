---
description: Architectural reference for CaveBlockPriorityModifier
---

# CaveBlockPriorityModifier

**Package:** com.hypixel.hytale.server.worldgen.cave
**Type:** Singleton

## Definition
```java
// Signature
public class CaveBlockPriorityModifier implements BlockPriorityModifier {
```

## Architecture & Concepts
The CaveBlockPriorityModifier is a stateless, rule-based component that implements the **Strategy** design pattern through the BlockPriorityModifier interface. Its sole responsibility is to resolve block placement conflicts during the server-side cave generation process.

Within the world generation pipeline, various algorithms (or "carvers") attempt to place blocks into a chunk. When a new block (the *target*) is placed next to an existing block (the *current*), this modifier is invoked to determine the final state of both blocks. It encapsulates a small, hard-coded set of rules that dictate which block type has priority or how they should transform when adjacent.

This component is critical for creating specific geological features or material transitions within cave systems, preventing abrupt or unnatural block boundaries. For example, it can enforce a rule where one mineral vein consumes another upon contact.

### Lifecycle & Ownership
- **Creation:** The single instance is created eagerly by the JVM during class loading, via the `public static final INSTANCE` field. This is a classic eager initialization singleton pattern.
- **Scope:** Application-scoped. The `INSTANCE` persists for the entire lifetime of the server process. It is globally accessible and shared across all world generation threads.
- **Destruction:** The object is eligible for garbage collection only when its class loader is unloaded, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields, and its methods are pure functions whose output depends exclusively on their input arguments.
- **Thread Safety:** The CaveBlockPriorityModifier is **inherently thread-safe**. As a stateless singleton, it can be safely invoked by numerous concurrent world generation threads without any risk of race conditions or need for external synchronization. This design is optimal for the highly parallelized nature of world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyCurrent(current, target) | byte | O(1) | Determines the resulting block ID for the *existing* block based on the new adjacent block. |
| modifyTarget(current, target) | byte | O(1) | Determines the resulting block ID for the *newly placed* block based on the existing adjacent block. |

## Integration Patterns

### Standard Usage
This class is not intended to be retrieved from a dependency injection context. It should be accessed directly via its static `INSTANCE` field from within a world generation algorithm, such as a cave carver.

```java
// Within a world generation algorithm...
byte existingBlock = chunk.getBlock(x, y, z);
byte blockToPlace = getCaveMaterial();

// Apply the modifier's rules
byte finalExistingBlock = CaveBlockPriorityModifier.INSTANCE.modifyCurrent(existingBlock, blockToPlace);
byte finalBlockToPlace = CaveBlockPriorityModifier.INSTANCE.modifyTarget(existingBlock, blockToPlace);

// Write the final block states to the chunk
chunk.setBlock(x, y, z, finalExistingBlock);
chunk.setBlock(newX, newY, newZ, finalBlockToPlace);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a new instance with `new CaveBlockPriorityModifier()`. This defeats the singleton pattern, creates unnecessary garbage, and offers no benefit, as the object is stateless. Always use `CaveBlockPriorityModifier.INSTANCE`.
- **Context-Unaware Usage:** The hard-coded block ID rules (e.g., 5, 6, 8) are tightly coupled to the cave generation context. Using this modifier in other world generation stages, such as surface biome placement, will cause incorrect and unpredictable block transformations if those IDs represent different materials.

## Data Pipeline
The CaveBlockPriorityModifier acts as a transformation step within the broader world generation data flow. It does not initiate I/O or events but rather mutates data in-flight.

> Flow:
> Cave Carver Algorithm -> Identifies adjacent block IDs -> **CaveBlockPriorityModifier** -> Returns modified block IDs -> Chunk Voxel Buffer Update


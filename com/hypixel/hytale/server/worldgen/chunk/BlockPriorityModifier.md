---
description: Architectural reference for BlockPriorityModifier
---

# BlockPriorityModifier

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface BlockPriorityModifier {
```

## Architecture & Concepts
The BlockPriorityModifier interface defines a contract for resolving block placement conflicts during the world generation process. It embodies the Strategy Pattern, allowing different world generation features (e.g., ore vein placement, cave carving, structure generation) to specify how they should interact with pre-existing blocks within a chunk buffer.

This system is fundamental to creating a layered and non-destructive world generation pipeline. When a generator attempts to place a block at a coordinate that is already occupied, an implementation of this interface is invoked to arbitrate the outcome. It determines whether the new block should overwrite the old one, be discarded, or if some other transformation should occur.

The static field BlockPriorityModifier.NONE is a default implementation following the Null Object Pattern. It provides a "pass-through" behavior where the existing block is never modified and the new block is always placed as-is, effectively disabling any conflict resolution logic.

## Lifecycle & Ownership
- **Creation:** Implementations of this interface are typically instantiated as stateless, transient objects by higher-level world generation orchestrators or feature placers. They are often defined as static singletons or created on-the-fly for a specific generation task.
- **Scope:** The lifecycle of a BlockPriorityModifier instance is tied to a specific world generation operation. It is passed into the relevant methods and is eligible for garbage collection as soon as the operation completes. It holds no long-term state.
- **Destruction:** Managed entirely by the Java Garbage Collector. As these objects are stateless and short-lived, there are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** This interface is designed to be implemented by **stateless** classes. An implementation should not contain any mutable fields or rely on external state to perform its calculations. Its output must be solely dependent on its input parameters.
- **Thread Safety:** Implementations **must be** thread-safe. The server's world generation engine is highly parallelized, processing multiple chunks and features concurrently. A stateful or non-thread-safe implementation of BlockPriorityModifier will introduce severe race conditions, leading to chunk corruption and unpredictable world generation. The contract implicitly requires implementations to be pure functions.

## API Surface
The API provides two distinct modification hooks to handle complex interaction logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyCurrent(byte current, byte target) | byte | O(1) | Determines the fate of the block *already present* in the chunk buffer. It can leave the current block untouched or transform it based on the incoming target block. |
| modifyTarget(byte original, byte target) | byte | O(1) | Determines the final form of the *new block* being placed. It can modify the target block based on the block that was originally at the location before the current generation step began. |
| NONE | BlockPriorityModifier | O(1) | A static, no-op implementation that performs no modification. `modifyCurrent` returns `current`, and `modifyTarget` returns `target`. |

## Integration Patterns

### Standard Usage
A world generation feature will receive or create a BlockPriorityModifier to control its placement behavior. It is used as a configuration parameter for the generation algorithm.

```java
// A feature placer uses a modifier to resolve conflicts
void placeFeature(ChunkBuffer buffer, BlockPriorityModifier modifier) {
    for (Position pos : featurePositions) {
        byte currentBlock = buffer.getBlock(pos);
        byte targetBlock = this.getFeatureBlock();

        // The modifier arbitrates the conflict
        byte finalTarget = modifier.modifyTarget(currentBlock, targetBlock);

        // Note: modifyCurrent is often handled internally by the buffer
        // when setting the final block, but is shown here for clarity.
        buffer.setBlock(pos, finalTarget);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating a modifier that maintains state (e.g., a counter) is a critical error. This will break when used in the multithreaded world generator, causing non-deterministic and corrupt chunk data.
- **Ignoring the NONE Constant:** Do not create new anonymous classes that replicate the behavior of BlockPriorityModifier.NONE. Always use the provided static instance for default, non-modifying behavior to reduce object allocation.

## Data Pipeline
This component sits directly within the server-side world generation pipeline, acting as a decision point for data being written into a chunk's block buffer.

> Flow:
> World Generation Feature -> Proposes a block (target) at a coordinate -> **BlockPriorityModifier** -> Resolves conflict with existing block (current) -> Returns final block ID -> Chunk Buffer Write


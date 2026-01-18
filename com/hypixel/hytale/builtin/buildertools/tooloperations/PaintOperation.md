---
description: Architectural reference for PaintOperation
---

# PaintOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class PaintOperation extends ToolOperation {
```

## Architecture & Concepts
The PaintOperation class is a stateful, short-lived command object that executes the logic for a "paint" or "re-materialize" action within the server-side Builder Tools framework. It is not a long-running service but rather a single-use executor that translates a player's intent, received via a network packet, into specific block changes in the world.

This class is instantiated to handle one specific tool application event. Its primary responsibility is to apply a material pattern to a block coordinate based on the player's current brush settings, most notably the *brush density*. It distinguishes between painting over existing blocks and placing new blocks in empty space, tracking the latter for potential optimizations or undos.

It operates as a delegate within a larger system. A generic shape-generation algorithm (e.g., for a sphere or cube) determines *which* blocks to affect, and an instance of PaintOperation determines *how* each of those blocks is affected.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level tool controller in response to a `BuilderToolOnUseInteraction` packet from a client. The controller is responsible for parsing the packet and providing the necessary context, such as the player entity and world edit session.
- **Scope:** The lifecycle of a PaintOperation instance is extremely brief, confined to the scope of a single tool usage event. It is created, its `execute0` method is invoked (potentially many times for all blocks in the target shape), and it is then immediately discarded.
- **Destruction:** The object becomes eligible for garbage collection as soon as the parent tool controller finishes processing the block modifications for the single user action. There are no persistent references to it.

## Internal State & Concurrency
- **State:** PaintOperation is stateful.
    - **brushDensity:** An integer representing the probability (0-100) that a block will be painted. This value is read from the player's tool settings upon construction and is immutable for the lifetime of the instance.
    - **packedPlacedBlockPositions:** A mutable `LongOpenHashSet` that accumulates the packed coordinates of blocks placed in empty air. This state is built up across multiple calls to `execute0` and is used by the builder tool system to track newly created geometry.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created and used exclusively within the context of a single thread that is processing a specific player's actions. The internal state, particularly the `packedPlacedBlockPositions` set, is not protected against concurrent modification. All synchronization is expected to be handled at a higher level, typically by the world or entity processing thread.

## API Surface
The primary interaction point is the `execute0` method, inherited and implemented from the `ToolOperation` superclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(1) | Executes the paint logic for a single block coordinate. Returns true to indicate the operation was processed. This method is not idempotent and modifies internal state. |

## Integration Patterns

### Standard Usage
A PaintOperation is never used in isolation. It is created by a tool handler and passed to a shape generator, which invokes `execute0` for each block within the generated volume.

```java
// Simplified conceptual example of a tool handler
void onPlayerUseTool(Player player, BuilderToolOnUseInteraction packet) {
    // 1. Create the operation based on player settings
    PaintOperation operation = new PaintOperation(worldRef, player, packet, accessor);

    // 2. Generate the shape (e.g., a sphere)
    BlockVolume sphere = ShapeGenerator.createSphere(packet.getOrigin(), packet.getRadius());

    // 3. Apply the operation to each block in the shape
    for (BlockPos pos : sphere) {
        operation.execute0(pos.getX(), pos.getY(), pos.getZ());
    }

    // 4. The operation is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to cache and reuse a PaintOperation instance for a subsequent tool action. Its internal state (`packedPlacedBlockPositions`) is specific to the single event for which it was created and will not be cleared.
- **Concurrent Execution:** Never call `execute0` from multiple threads on the same PaintOperation instance. This will cause race conditions on its internal state and likely corrupt the world edit session.

## Data Pipeline
The class acts as a transformation stage in the data pipeline that begins with player input and ends with a modified world state.

> Flow:
> Client Input -> `BuilderToolOnUseInteraction` Packet -> Server Network Layer -> Builder Tool Controller -> **PaintOperation Instantiation** -> `execute0` -> World Edit Session -> World Storage Commit


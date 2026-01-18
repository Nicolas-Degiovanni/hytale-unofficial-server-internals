---
description: Architectural reference for SculptOperation
---

# SculptOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class SculptOperation extends ToolOperation {
```

## Architecture & Concepts
The SculptOperation class is a concrete implementation of the `ToolOperation` command pattern, central to the server-side Builder Tools framework. It encapsulates the logic for a single "sculpt" action, which involves adding or removing voxels within a defined brush area to create organic shapes.

This class acts as the final execution unit that translates a player's network request into tangible world modifications. It is instantiated by a higher-level tool handler in direct response to a `BuilderToolOnUseInteraction` packet from the client. Its sole responsibility is to apply the sculpting logic for a single block coordinate, taking into account brush settings, interaction type (add/remove/smooth), and the local block environment.

Unlike persistent services, a SculptOperation is a short-lived, single-use object that holds the context for one specific point within a larger brush application.

### Lifecycle & Ownership
- **Creation:** Instantiated by the Builder Tools framework when a player uses a tool configured for a sculpt operation. The constructor is passed the player, the interaction packet, and an accessor to the entity-component system, providing all necessary context for the operation.
- **Scope:** Extremely short-lived. An instance of SculptOperation exists only for the duration of processing a single block coordinate within the brush's area of effect. It is created, its `execute0` method is called, and it is then immediately eligible for garbage collection.
- **Destruction:** The object is automatically garbage collected once the calling method, typically a loop iterating over the brush volume, moves to the next coordinate and drops the reference. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** SculptOperation is stateful, but its state is scoped exclusively to the single interaction it represents. It caches calculated values from the player's tool settings, such as `smoothVolume`, `smoothRadius`, and `brushDensity`, upon construction. It also holds a reference to a `LongOpenHashSet`, `packedPlacedBlockPositions`, to prevent newly placed blocks from being immediately affected by subsequent operations within the same brush stroke. This state is **Mutable** but confined to its brief lifecycle.

- **Thread Safety:** This class is **Not Thread-Safe** and must not be used concurrently. It is designed to be created, used, and discarded within the server's main world-tick thread. It directly manipulates a `WorldEdit` buffer without any locking mechanisms. Any attempt to access it from multiple threads will result in world corruption, race conditions, and undefined behavior.

## API Surface
The primary contract is the `execute0` method inherited from its parent, `ToolOperation`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(1) | Executes the sculpt logic for a single block coordinate. Reads surrounding block data and writes a new material to the active `WorldEdit` buffer. The logic branches based on the interaction type and tool settings provided at construction. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is instantiated and managed entirely by the Builder Tools system. The conceptual flow involves a manager iterating over a volume and applying the operation at each point.

```java
// Conceptual example of how the framework uses this class.
// Do not replicate this pattern.

ToolOperation operation = new SculptOperation(ref, player, packet, accessor);
BrushVolume brush = getBrushVolumeForPacket(packet);

for (BlockPos pos : brush) {
    operation.execute0(pos.getX(), pos.getY(), pos.getZ());
}
// The 'operation' object is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct a SculptOperation using `new`. The Builder Tools framework is responsible for its creation and lifecycle. Bypassing the framework will lead to an uninitialized and non-functional object.
- **State Reuse:** Do not retain a reference to a SculptOperation instance after its initial use. The object is configured for a single interaction packet and is not designed to be reused for subsequent tool actions.
- **Concurrent Execution:** Never invoke `execute0` from a separate thread. All world modifications must be synchronized with the main server tick, and this class performs direct, unsynchronized edits to a world buffer.

## Data Pipeline
The SculptOperation is a critical stage in the server-side processing of a player's building action. It sits between the network protocol and the world state modification layer.

> Flow:
> Player Client Input -> `BuilderToolOnUseInteraction` Packet -> Server Network Layer -> Builder Tools Framework -> **SculptOperation** Instantiation -> `execute0` call -> `WorldEdit` Buffer Modification -> World State Commit -> Voxel Data sent to Clients

